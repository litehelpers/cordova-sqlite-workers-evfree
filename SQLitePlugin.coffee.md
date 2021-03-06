# SQLite plugin in Markdown (litcoffee)

    ###
    License for this version: GPL v3 (http://www.gnu.org/licenses/gpl.txt) or commercial license.
    Contact for commercial license: info@litehelpers.net
    ###

#### Use coffee compiler to compile this directly into Javascript

# Top-level SQLite plugin objects

## root window object:

    root = @

## constant(s):

    READ_ONLY_REGEX = /^\s*(?:drop|delete|insert|update|create)\s/i

    # per-db state
    DB_STATE_INIT = "INIT"
    DB_STATE_OPEN = "OPEN"

    ###
    OPTIONAL: Transaction SQL chunking
    MAX_SQL_CHUNK is adjustable, set to 0 (or -1) to disable chunking
    ###
    MAX_SQL_CHUNK = 0

    MAX_PART_SIZE = 500

## global(s):

    isWorker = !!!root.document

    # per-db map of locking and queueing
    # XXX NOTE: This is NOT cleaned up when a db is closed and/or deleted.
    # If the record is simply removed when a db is closed or deleted,
    # it will cause some test failures and may break large-scale
    # applications that repeatedly open and close the database.
    # [BUG #210] TODO: better to abort and clean up the pending transaction state.
    # XXX TBD this will be renamed and include some more per-db state.
    # NOTE: In case txLocks is renamed or replaced the selfTest has to be adapted as well.
    txLocks = {}

    # XXX NOTE: NEVER expected to be false for a database in this version, TBD remove:
    useflatjson = false

    if isWorker
      importScripts 'aqworker.js'
      aqsetcbprefix 'sqlcb'

## utility functions:

    # Errors returned to callbacks must conform to `SqlError` with a code and message.
    # Some errors are of type `Error` or `string` and must be converted.
    newSQLError = (error, code) ->
      sqlError = error
      code = 0 if !code # unknown by default

      if !sqlError
        sqlError = new Error "a plugin had an error but provided no response"
        sqlError.code = code

      if typeof sqlError is "string"
        sqlError = new Error error
        sqlError.code = code

      if !sqlError.code && sqlError.message
        sqlError.code = code

      if !sqlError.code && !sqlError.message
        sqlError = new Error "an unknown error was returned: " + JSON.stringify(sqlError)
        sqlError.code = code

      return sqlError

    nextTick = if isWorker then (fun) ->
      # XXX FUTURE TBD use setImmediate (??)
      setTimeout(fun, 0)
      return
    else window.setImmediate || (fun) ->
      window.setTimeout(fun, 0)
      return

    ###
      Utility that avoids leaking the arguments object. See
      https://www.npmjs.org/package/argsarray
    ###
    argsArray = (fun) ->
      return ->
        len = arguments.length
        if len
          args = []
          i = -1
          while ++i < len
            args[i] = arguments[i]
          return fun.call this, args
        else
          return fun.call this, []

## SQLite plugin db-connection handle

#### NOTE: there can be multipe SQLitePlugin db-connection handles per open db.

#### SQLite plugin db connection handle object is defined by a constructor function and prototype member functions:

    SQLitePlugin = (openargs, openSuccess, openError) ->
      # console.log "SQLitePlugin openargs: #{JSON.stringify openargs}"

      # SHOULD already be checked by openDatabase:
      if !(openargs and openargs['name'])
        throw newSQLError "Cannot create a SQLitePlugin db instance without a db name"

      dbname = openargs.name

      if typeof dbname != 'string'
        throw newSQLError 'sqlite plugin database name must be a string'

      @openargs = openargs
      @dbname = dbname

      @openSuccess = openSuccess
      @openError = openError

      @openSuccess or
        @openSuccess = ->
          console.log "DB opened: " + dbname
          return

      @openError or
        @openError = (e) ->
          console.log e.message
          return

      @open @openSuccess, @openError
      return

    SQLitePlugin::databaseFeatures = isSQLitePluginDatabase: true

    # Keep track of state of open db connections
    # XXX FUTURE TBD this *may* be moved and renamed,
    # or even combined with txLocks if possible.
    # NOTE: In case txLocks is renamed or replaced the selfTest has to be adapted as well.
    SQLitePlugin::openDBs = {}

    SQLitePlugin::addTransaction = (t) ->
      if !txLocks[@dbname]
        txLocks[@dbname] = {
          queue: []
          inProgress: false
        }
      txLocks[@dbname].queue.push t
      if @dbname of @openDBs && @openDBs[@dbname] isnt DB_STATE_INIT
        # FUTURE TBD: rename startNextTransaction to something like
        # triggerTransactionQueue
        # ALT TBD: only when queue has length of 1 (and test)??
        @startNextTransaction()

      else
        if @dbname of @openDBs
          console.log 'new transaction is queued, waiting for open operation to finish'
        else
          # XXX SHOULD NOT GET HERE.
          # FUTURE TBD TODO: in this exceptional case abort and discard the transaction.
          console.log 'database is closed, new transaction is [stuck] waiting until db is opened again!'
      return

    SQLitePlugin::beginTransaction = (error) ->
      if !@openDBs[@dbname]
        throw newSQLError 'database not open'

      myfn = (tx) -> return
      mytx = new SQLitePluginTransaction(this, myfn, error, null, false, false)
      mytx.canPause = true
      mytx.addStatement "BEGIN", [], null, (tx, err) ->
        throw newSQLError "unable to begin transaction: " + err.message, err.code
      mytx.txlock = true
      @addTransaction mytx

      mytx

    SQLitePlugin::transaction = (fn, error, success) ->
      # FUTURE TBD check for valid fn here
      if !@openDBs[@dbname]
        error newSQLError 'database not open'
        return

      @addTransaction new SQLitePluginTransaction(this, fn, error, success, true, false)
      return

    SQLitePlugin::readTransaction = (fn, error, success) ->
      # FUTURE TBD check for valid fn here (and add test for this)
      if !@openDBs[@dbname]
        error newSQLError 'database not open'
        return

      @addTransaction new SQLitePluginTransaction(this, fn, error, success, false, true)
      return

    SQLitePlugin::startNextTransaction = ->
      self = @

      nextTick =>
        if !(@dbname of @openDBs) || @openDBs[@dbname] isnt DB_STATE_OPEN
          console.log 'cannot start next transaction: database not open'
          return

        txLock = txLocks[self.dbname]
        if !txLock
          console.log 'cannot start next transaction: database connection is lost'
          # XXX TBD TODO (BUG #210/??): abort all pending transactions with error cb [and test!!]
          # @abortAllPendingTransactions()
          return

        else if txLock.queue.length > 0 && !txLock.inProgress
          # start next transaction in q
          txLock.inProgress = true
          txLock.queue.shift().start()
        return

      return

    SQLitePlugin::abortAllPendingTransactions = ->
      # extra debug info:
      # if txLocks[@dbname] then console.log 'abortAllPendingTransactions with transaction queue length: ' + txLocks[@dbname].queue.length
      # else console.log 'abortAllPendingTransactions with no transaction lock state'

      txLock = txLocks[@dbname]
      if !!txLock && txLock.queue.length > 0
        # XXX TODO: what to do in case there is a (stray) transaction in progress?
        #console.log 'abortAllPendingTransactions - cleanup old transaction(s)'
        for tx in txLock.queue
          tx.abortFromQ newSQLError 'Invalid database handle'

        # XXX TODO: consider cleaning up (delete) txLocks[@dbname] resource,
        # in case it is known there are no more pending transactions
        txLock.queue = []
        txLock.inProgress = false

      return

    SQLitePlugin::open = (success, error) ->
      if @dbname of @openDBs
        console.log 'database already open: ' + @dbname

        # for a re-open run the success cb async so that the openDatabase return value
        # can be used in the success handler as an alternative to the handler's
        # db argument
        nextTick =>
          success @
          return

        # (done)

      else
        console.log 'OPEN database: ' + @dbname

        opensuccesscb = (a1) =>
          # NOTE: the db state is NOT stored (in @openDBs) if the db was closed or deleted.
          console.log 'OPEN database: ' + @dbname + ' OK'

          # Needed to distinguish between Android version (with flat JSON batch sql interface) and
          # other versions (JSON batch interface unchanged)
          if !!a1 and a1 == 'a1'
            console.log 'Detected Android/iOS version with flat JSON interface'
            useflatjson = true

          #if !@openDBs[@dbname] then call open error cb, and abort pending tx if any
          if !@openDBs[@dbname]
            console.log 'database was closed during open operation'
            # XXX TODO (WITH TEST) ref BUG litehelpers/Cordova-sqlite-storage#210:
            # if !!error then error newSQLError 'database closed during open operation'
            # @abortAllPendingTransactions()

          if @dbname of @openDBs
            @openDBs[@dbname] = DB_STATE_OPEN

          if !!success then success @

          txLock = txLocks[@dbname]
          if !!txLock && txLock.queue.length > 0 && !txLock.inProgress
            @startNextTransaction()
          return

        openerrorcb = =>
          console.log 'OPEN database: ' + @dbname + ' failed, aborting any pending transactions'
          # XXX TODO: newSQLError missing the message part!
          if !!error then error newSQLError 'Could not open database'
          delete @openDBs[@dbname]
          @abortAllPendingTransactions()
          return

        # store initial DB state:
        @openDBs[@dbname] = DB_STATE_INIT

        # XXX NO UNWANTED WORKAROUND SOLUTION for
        # BUG litehelpers/Cordova-sqlite-storage#666
        # in this plugin version as discussed in
        # https://github.com/litehelpers/cordova-sqlite-evmax-ext-workers-legacy-build-free/issues/7
        # nextTick =>
        if (true)
          if isWorker
            aqrequest 'sq', 'open', (encodeURIComponent (JSON.stringify [@openargs])), (s) ->
              if s == 'a1'
                opensuccesscb s
              else
                openerrorcb()
          else
            root.sqlitePluginHelper.exec 'open', [ @openargs ], opensuccesscb, openerrorcb

      return

    SQLitePlugin::close = (success, error) ->
      if @dbname of @openDBs
        if txLocks[@dbname] && txLocks[@dbname].inProgress
          # FUTURE TBD TODO ref BUG litehelpers/Cordova-sqlite-storage#210:
          # Wait for current tx to finish then close,
          # then abort any other pending transactions
          # (and cleanup any other internal resources).
          # (This would need testing!!)
          console.log 'cannot close: transaction is in progress'
          error newSQLError 'database cannot be closed while a transaction is in progress'
          return

        console.log 'CLOSE database: ' + @dbname

        # NOTE: closing one db handle disables other handles to same db
        # FUTURE TBD TODO ref litehelpers/Cordova-sqlite-storage#210:
        # Add a dispose method to simply invalidate the
        # current database object ("this")
        delete @openDBs[@dbname]

        if txLocks[@dbname] then console.log 'closing db with transaction queue length: ' + txLocks[@dbname].queue.length
        else console.log 'closing db with no transaction lock state'

        # XXX TODO BUG litehelpers/Cordova-sqlite-storage#210:
        # abort all pending transactions (with error callback)
        # when closing a database (needs testing!!)
        # (and cleanup any other internal resources)

        root.sqlitePluginHelper.exec 'close', [ { path: @dbname } ], success, error

      else
        console.log 'cannot close: database is not open'
        if error then nextTick -> error()

      return

    SQLitePlugin::executeSql = (statement, params, success, error) ->
      # XXX TODO: better to capture the result, and report it once
      # the transaction has completely finished.
      # This would fix BUG #204 (cannot close db in db.executeSql() callback).
      mysuccess = (t, r) -> if !!success then success r
      myerror = (t, e) -> if !!error then error e

      myfn = (tx) ->
        tx.addStatement(statement, params, mysuccess, myerror)
        return

      @addTransaction new SQLitePluginTransaction(this, myfn, null, null, false, false)
      return

    SQLitePlugin::sqlBatch = (sqlStatements, success, error) ->
      if !sqlStatements || sqlStatements.constructor isnt Array
        throw newSQLError 'sqlBatch expects an array'

      batchList = []

      for st in sqlStatements
        if st.constructor is Array
          if st.length == 0
            throw newSQLError 'sqlBatch array element of zero (0) length'

          batchList.push
            sql: st[0]
            params: if st.length == 0 then [] else st[1]

        else
          batchList.push
            sql: st
            params: []

      myfn = (tx) ->
        for elem in batchList
          tx.addStatement(elem.sql, elem.params, null, null)

      @addTransaction new SQLitePluginTransaction(this, myfn, error, success, true, false)
      return

## SQLite plugin transaction object for batching:

    SQLitePluginTransaction = (db, fn, error, success, txlock, readOnly) ->
      # FUTURE TBD check this earlier:
      if typeof(fn) != "function"
        ###
        This is consistent with the implementation in Chrome -- it
        throws if you pass anything other than a function. This also
        prevents us from stalling our txQueue if somebody passes a
        false value for fn.
        ###
        throw newSQLError "transaction expected a function"

      @db = db
      @fn = fn
      @error = error
      @success = success
      @txlock = txlock
      @readOnly = readOnly
      @canPause = false
      @isPaused = false
      @executes = []

      if txlock
        @addStatement "BEGIN", [], null, (tx, err) ->
          throw newSQLError "unable to begin transaction: " + err.message, err.code

      # Workaround for litehelpers/Cordova-sqlite-storage#409
      # extra statement in case user function does not add any SQL statements
      else
        @addStatement "SELECT 1", [], null, null

      return

    SQLitePluginTransaction::start = ->
      try
        @fn this

        if @executes.length > 0
          @run()

      catch err
        # If "fn" throws, we must report the whole transaction as failed.
        txLocks[@db.dbname].inProgress = false
        @db.startNextTransaction()
        if @error
          @error newSQLError err

      return

    SQLitePluginTransaction::executeSql = (sql, values, success, error) ->

      if @finalized
        throw {message: 'InvalidStateError: DOM Exception 11: This transaction is already finalized. Transactions are committed after its success or failure handlers are called. If you are using a Promise to handle callbacks, be aware that implementations following the A+ standard adhere to run-to-completion semantics and so Promise resolution occurs on a subsequent tick and therefore after the transaction commits.', code: 11}
        return

      if @readOnly && READ_ONLY_REGEX.test(sql)
        @handleStatementFailure(error, {message: 'invalid sql for a read-only transaction'})
        return

      @addStatement(sql, values, success, error)

      if @isPaused
        @isPaused = false
        @run()

      return

    SQLitePluginTransaction::end = (success, error) ->
      if !@canPause
        throw newSQLError 'Sorry invalid usage'

      @canPause = false
      @success = success
      @error = error
      if @isPaused
        @isPaused = false
        if @executes.length == 0
          @$finish()
        else
          @run()

      return

    SQLitePluginTransaction::abort = (errorcb) ->
      if !@canPause
        throw newSQLError 'Sorry invalid usage'

      @canPause = false
      @error = errorcb
      @addStatement 'INVALID STATEMENT', [], null, null

      if @isPaused
        @isPaused = false
        @run()

      return

    # This method adds the SQL statement to the transaction queue but does not check for
    # finalization since it is used to execute COMMIT and ROLLBACK.
    SQLitePluginTransaction::addStatement = (sql, values, success, error) ->

      params = []
      if !!values && values.constructor == Array
        for v in values
          t = typeof v
          params.push (
            if v == null || v == undefined || t == 'number' || t == 'string' then v
            else if v instanceof Blob then v.valueOf()
            else v.toString()
          )

      @executes.push
        success: success
        error: error

        sql: sql
        params: params

      if MAX_SQL_CHUNK > 0 && @executes.length > MAX_SQL_CHUNK
        @run()

      return

    SQLitePluginTransaction::handleStatementSuccess = (handler, response) ->
      if !handler
        return

      rows = response.rows || []
      payload =
        rows:
          item: (i) ->
            rows[i]

          length: rows.length

        rowsAffected: response.rowsAffected or 0
        insertId: response.insertId or undefined

      handler this, payload

      return

    SQLitePluginTransaction::handleStatementFailure = (handler, response) ->
      if !handler
        throw newSQLError "a statement with no error handler failed: " + response.message, response.code
      if handler(this, response) isnt false
        throw newSQLError "a statement error callback did not return false: " + response.message, response.code
      return

    SQLitePluginTransaction::run = ->
      # persist for handlerFor callbacks:
      txFailure = null
      # sql statements from queue:
      batchExecutes = @executes
      # NOTE: If this is zero it will not work. Workaround is applied in the constructor.
      # FUTURE TBD: It would be better to fix the problem here.
      waiting = batchExecutes.length
      @executes = []
      # my tx object [this]
      tx = @

      handlerFor = (index, didSucceed) ->
        (response) ->
          try
            if didSucceed
              tx.handleStatementSuccess batchExecutes[index].success, response
            else
              sqlError = newSQLError(response)
              if !!response.result
                sqlError.code = response.result.code
                sqlError.sqliteCode = response.result.sqliteCode
              tx.handleStatementFailure batchExecutes[index].error, sqlError
          catch err
            if !txFailure
              txFailure = newSQLError(err)

          if --waiting == 0
            if txFailure
              tx.$abort txFailure
            else if tx.executes.length > 0
              # new requests have been issued by the callback
              # handlers, so run another batch.
              tx.run()
            else if tx.canPause
              tx.isPaused = true
            else
              tx.$finish()

          return

      @run_batch batchExecutes, handlerFor

      return

    batchid = 0

    SQLitePluginTransaction::run_batch = (batchExecutes, handlerFor) ->
      flatlist = []
      mycbmap = {}

      i = 0
      while i < batchExecutes.length
        request = batchExecutes[i]

        mycbmap[i] =
          success: handlerFor(i, true)
          error: handlerFor(i, false)

        flatlist.push request.sql
        flatlist.push request.params.length
        for p in request.params
          flatlist.push p

        i++

      mycb = (result) ->
        i = 0
        ri = 0
        rl = result.length

        while ri < rl
          r = result[ri++]
          q = mycbmap[i]

          if r == 'ok'
            q.success { rows: [] }

          else if r is "ch2"
            changes = result[ri++]
            insert_id = result[ri++]
            q.success
              rowsAffected: changes
              insertId: insert_id

          else if r == 'okrows'
            rows = []
            changes = 0
            insert_id = undefined

            if result[ri] == 'changes'
              ++ri
              changes = result[ri++]

            if result[ri] == 'insert_id'
              ++ri
              insert_id = result[ri++]

            while result[ri] != 'endrows'
              c = result[ri++]
              j = 0
              row = {}

              while j < c
                k = result[ri++]
                v = result[ri++]
                row[k] = v
                ++j

              rows.push row

            q.success { rows: rows, rowsAffected: changes, insertId: insert_id }
            ++ri

          else if r == 'error'
            code = result[ri++]
            sqliteCode = result[ri++]
            errormessage = result[ri++]
            q.error
              result:
                code: code
                sqliteCode: sqliteCode
                message: errormessage

          ++i

        return

      if isWorker
        ++batchid
        batchname = aqcbhandlername + '.' + batchid
        rem = flatlist
        flatlist = null
        aqrequest 'sq', 'batchStart', encodeURIComponent(JSON.stringify([ {
          dbargs: dbname: @db.dbname
          batchid: batchname
          flen: batchExecutes.length
        } ])), (s) -> return

        k = 0
        partid = 0
        while rem.length > 0
          rlength = rem.length
          pl = if rlength > MAX_PART_SIZE then MAX_PART_SIZE else rlength
          part = rem.slice(0, pl)
          rem = rem.slice(pl)
          #console.log 'part: ' + JSON.stringify(part)
          #console.log 'rem: ' + JSON.stringify(rem)
          aqrequest 'sq', 'batchPart', encodeURIComponent(JSON.stringify([ {
            batchid: batchname
            partid: ++partid
            flen: batchExecutes.length
            part: part
          } ])), (s) -> return

        aqrequest 'sq', 'batchRun', encodeURIComponent(JSON.stringify([ { batchid: batchname } ])), (s) ->
          json = decodeURIComponent(s)
          res = JSON.parse(json)
          mycb res
          return

      else
        root.sqlitePluginHelper.exec 'backgroundExecuteSqlBatch',
          [{dbargs: {dbname: @db.dbname}, flen: batchExecutes.length, flatlist: flatlist}],
          mycb, null

      return

    SQLitePluginTransaction::$abort = (txFailure) ->
      if @finalized then return
      tx = @

      succeeded = (tx) ->
        txLocks[tx.db.dbname].inProgress = false
        tx.db.startNextTransaction()
        if tx.error then tx.error txFailure
        return

      failed = (tx, err) ->
        txLocks[tx.db.dbname].inProgress = false
        tx.db.startNextTransaction()
        if tx.error then tx.error newSQLError("error while trying to roll back: " + err.message, err.code)
        return

      @finalized = true

      if @txlock
        @addStatement "ROLLBACK", [], succeeded, failed
        @run()
      else
        succeeded(tx)

      return

    SQLitePluginTransaction::$finish = ->
      if @finalized then return
      tx = @

      succeeded = (tx) ->
        txLocks[tx.db.dbname].inProgress = false
        tx.db.startNextTransaction()
        if tx.success then tx.success()
        return

      failed = (tx, err) ->
        txLocks[tx.db.dbname].inProgress = false
        tx.db.startNextTransaction()
        if tx.error then tx.error newSQLError("error while trying to commit: " + err.message, err.code)
        return

      @finalized = true

      if @txlock
        @addStatement "COMMIT", [], succeeded, failed
        @run()
      else
        succeeded(tx)

      return

    SQLitePluginTransaction::abortFromQ = (sqlerror) ->
      # NOTE: since the transaction is waiting in the queue,
      # the transaction function containing the SQL statements
      # would not be run yet. Simply report the transaction error.
      if @error
        @error sqlerror

      return

## SQLite plugin object factory:

    # OLD:
    dblocations = [ "docs", "libs", "nosync" ]

    iosLocationMap =
      'default' : 'nosync'
      'Documents' : 'docs'
      'Library' : 'libs'

    SQLiteFactory =
      ###
      NOTE: this function should NOT be translated from Javascript
      back to CoffeeScript by js2coffee.
      If this function is edited in Javascript then someone will
      have to translate it back to CoffeeScript by hand.
      ###
      openDatabase: argsArray (args) ->
        if args.length < 1 || !args[0]
          throw newSQLError 'Sorry missing mandatory open arguments object in openDatabase call'

        #first = args[0]
        #openargs = null
        #okcb = null
        #errorcb = null

        #if first.constructor == String
        #  openargs = {name: first}

        #  if args.length >= 5
        #    okcb = args[4]
        #    if args.length > 5 then errorcb = args[5]

        #else
        #  openargs = first

        #  if args.length >= 2
        #    okcb = args[1]
        #    if args.length > 2 then errorcb = args[2]

        #dblocation = if !!openargs.location then dblocations[openargs.location] else null
        #openargs.dblocation = dblocation || dblocations[0]

        openargs = args[0]

        # check here
        if !openargs.name
          throw newSQLError 'Database name value is missing in openDatabase call'

        if !openargs.iosDatabaseLocation and !openargs.location and openargs.location isnt 0
          throw newSQLError 'Database location or iosDatabaseLocation value is now mandatory in openDatabase call'

        if !!openargs.location and !!openargs.iosDatabaseLocation
          throw newSQLError 'Ambiguous: both location or iosDatabaseLocation values are present in openDatabase call'

        dblocation =
          if !!openargs.location and openargs.location is 'default'
            iosLocationMap['default']
          else if !!openargs.iosDatabaseLocation
            iosLocationMap[openargs.iosDatabaseLocation]
          else
            dblocations[openargs.location]

        if !dblocation
          throw newSQLError 'Valid iOS database location could not be determined in openDatabase call'

        openargs.dblocation = dblocation

        if !!openargs.createFromLocation and openargs.createFromLocation == 1
          openargs.createFromResource = "1"

        if !!openargs.androidDatabaseImplementation and openargs.androidDatabaseImplementation == 2
          openargs.androidOldDatabaseImplementation = 1

        if !!openargs.androidLockWorkaround and openargs.androidLockWorkaround == 1
          openargs.androidBugWorkaround = 1

        okcb = null
        errorcb = null
        if args.length >= 2
          okcb = args[1]
          if args.length > 2 then errorcb = args[2]

        new SQLitePlugin openargs, okcb, errorcb

      deleteDatabase: (first, success, error) ->
        # XXX TODO BUG litehelpers/Cordova-sqlite-storage#367:
        # abort all pending transactions (with error callback)
        # when deleting a database
        # (and cleanup any other internal resources)
        # NOTE: This should properly close the database
        # (at least on the JavaScript side) before deleting.
        args = {}

        if first.constructor == String
          #console.log "delete db name: #{first}"
          #args.path = first
          #args.dblocation = dblocations[0]
          throw newSQLError 'Sorry first deleteDatabase argument must be an object'

        else
          #console.log "delete db args: #{JSON.stringify first}"
          if !(first and first['name']) then throw new Error "Please specify db name"

          #args.path = first.name
          #dblocation = if !!first.location then dblocations[first.location] else null
          #args.dblocation = dblocation || dblocations[0]

          dbname = first.name

          if typeof dbname != 'string'
            throw newSQLError 'delete database name must be a string'

          args.path = dbname
          #dblocation = if !!first.location then dblocations[first.location] else null
          #args.dblocation = dblocation || dblocations[0]

        if !first.iosDatabaseLocation and !first.location and first.location isnt 0
          throw newSQLError 'Database location or iosDatabaseLocation value is now mandatory in deleteDatabase call'

        if !!first.location and !!first.iosDatabaseLocation
          throw newSQLError 'Ambiguous: both location or iosDatabaseLocation values are present in deleteDatabase call'

        dblocation =
          if !!first.location and first.location is 'default'
            iosLocationMap['default']
          else if !!first.iosDatabaseLocation
            iosLocationMap[first.iosDatabaseLocation]
          else
            dblocations[first.location]

        if !dblocation
          throw newSQLError 'Valid iOS database location could not be determined in deleteDatabase call'

        args.dblocation = dblocation

        # XXX TODO BUG litehelpers/Cordova-sqlite-storage#367 (repeated here):
        # abort all pending transactions (with error callback)
        # when deleting a database
        # (and cleanup any other internal resources)
        delete SQLitePlugin::openDBs[args.path]
        root.sqlitePluginHelper.exec 'delete', [ args ], success, error

        return

## Self test:

    SelfTest =
      DBNAME: '___$$$___litehelpers___$$$___test___$$$___.db'

      start: (successcb, errorcb) ->
        SQLiteFactory.deleteDatabase {name: SelfTest.DBNAME, location: 'default'},
          (-> SelfTest.step1(successcb, errorcb)),
          (-> SelfTest.step1(successcb, errorcb))
        return

      step1: (successcb, errorcb) ->
        SQLiteFactory.openDatabase {name: SelfTest.DBNAME, location: 'default'}, (db) ->
          check1 = false
          db.transaction (tx) ->
            tx.executeSql 'SELECT UPPER("Test") AS upperText', [], (ignored, resutSet) ->
              if !resutSet.rows
                return SelfTest.finishWithError errorcb, 'Missing resutSet.rows'

              if !resutSet.rows.length
                return SelfTest.finishWithError errorcb, 'Missing resutSet.rows.length'

              if resutSet.rows.length isnt 1
                return SelfTest.finishWithError errorcb,
                  "Incorrect resutSet.rows.length value: #{resutSet.rows.length} (expected: 1)"

              if !resutSet.rows.item(0).upperText
                return SelfTest.finishWithError errorcb,
                  'Missing resutSet.rows.item(0).upperText'

              if resutSet.rows.item(0).upperText isnt 'TEST'
                return SelfTest.finishWithError errorcb,
                  "Incorrect resutSet.rows.item(0).upperText value: #{resutSet.rows.item(0).upperText} (expected: 'TEST')"

              check1 = true
              return

            , (ignored, tx_sql_err) ->
              return SelfTest.finishWithError errorcb, "TX SQL error: #{tx_sql_err}"

            return

          , (tx_err) ->
            return SelfTest.finishWithError errorcb, "TRANSACTION error: #{tx_err}"

          , () ->
            # tx success:
            if !check1
              return SelfTest.finishWithError errorcb,
                'Did not get expected upperText result data'

            # SIMULATE SCENARIO IN BUG litehelpers/Cordova-sqlite-storage#666:
            db.executeSql 'BEGIN', null, (ignored) -> nextTick -> # (nextTick needed for Windows)
              # DELETE INTERNAL STATE to simulate the effects of location refresh or change:
              delete db.openDBs[SelfTest.DBNAME]
              delete txLocks[SelfTest.DBNAME]
              nextTick ->
                # VERIFY INTERNAL STATE IS DELETED:
                db.transaction (tx2) ->
                  tx2.executeSql 'SELECT 1'
                  return
                , (tx_err) ->
                  # EXPECTED RESULT:
                  if !tx_err
                    return SelfTest.finishWithError errorcb, 'Missing error object'
                  SelfTest.step2 successcb, errorcb
                  return
                , () ->
                  # NOT EXPECTED:
                  return SelfTest.finishWithError errorcb, 'Missing error object'
                return
              return

            return
          return

        , (open_err) ->
          SelfTest.finishWithError errorcb, "Open database error: #{open_err}"
        return

      step2: (successcb, errorcb) ->
        SQLiteFactory.openDatabase {name: SelfTest.DBNAME, location: 'default'}, (db) ->
          # TX SHOULD SUCCEED to demonstrate solution to BUG litehelpers/Cordova-sqlite-storage#666:
          db.transaction (tx) ->
            tx.executeSql 'SELECT ? AS myResult', [null], (ignored, resutSet) ->
              # XXX TBD IGNORED FOR NOW:
              # if !resutSet.rows
              #   return SelfTest.finishWithError errorcb, 'Missing resutSet.rows'
              # if !resutSet.rows.length
              #   return SelfTest.finishWithError errorcb, 'Missing resutSet.rows.length'
              # if resutSet.rows.length isnt 1
              #   return SelfTest.finishWithError errorcb,
              #     "Incorrect resutSet.rows.length value: #{resutSet.rows.length} (expected: 1)"
              # SelfTest.step3 successcb, errorcb
              return
            return

          , (txError) ->
            # XXX
            # EXPECTED RESULT DUE TO BUG litehelpers/Cordova-sqlite-storage#666:
            if !txError
              return SelfTest.finishWithError errorcb, 'Missing txError object'
            # second try should work:
            db.transaction (tx2) ->
              tx2.executeSql 'SELECT ? AS myResult', [null], (ignored, resutSet) ->
                if !resutSet.rows
                  return SelfTest.finishWithError errorcb, 'Missing resutSet.rows'
                if !resutSet.rows.length
                  return SelfTest.finishWithError errorcb, 'Missing resutSet.rows.length'
                if resutSet.rows.length isnt 1
                  return SelfTest.finishWithError errorcb,
                SelfTest.step3 successcb, errorcb
                return
              return
            , (tx2_err) ->
              return SelfTest.finishWithError errorcb, "UNEXPECTED TRANSACTION ERROR: #{tx2_err}"
            return

          , () ->
            # XXX
            # TX SUCCESS POSSIBLE FOR Android (android.database) ONLY
            # NOTE: Windows 10 (UWP) mobile platform also shows "Android" in navigator.userAgent,
            # filtered out here.
            # FUTURE TBD android.database implementation should be fixed to report error in this case.
            # XXX GONE:
            # if /Android/.test(navigator.userAgent) and not /Windows /.test(navigator.userAgent)
            #   return SelfTest.step3 successcb, errorcb
            # OTHERWISE:
            # TX SUCCESS NOT EXPECTED DUE TO BUG litehelpers/Cordova-sqlite-storage#666:
            return SelfTest.finishWithError errorcb, 'UNEXPECTED SUCCESS ref: litehelpers/Cordova-sqlite-storage#666'
          return

        , (open_err) ->
          SelfTest.finishWithError errorcb, "Open database error: #{open_err}"
        return

      step3: (successcb, errorcb) ->
        SQLiteFactory.openDatabase {name: SelfTest.DBNAME, location: 'default'}, (db) ->
          # FUTURE TBD TEST CRUD OPERATIONS (already fixed in a newer version branch)
          db.sqlBatch [
            'CREATE TABLE TestTable(TestColumn);'
            [ 'INSERT INTO TestTable (TestColumn) VALUES (?);', ['test-value'] ]
          ], () ->
            db.executeSql 'SELECT * FROM TestTable', [], (resutSet) ->
              if !resutSet.rows
                SelfTest.finishWithError errorcb, 'Missing resutSet.rows'
                return

              if !resutSet.rows.length
                SelfTest.finishWithError errorcb, 'Missing resutSet.rows.length'
                return

              if resutSet.rows.length isnt 1
                SelfTest.finishWithError errorcb,
                  "Incorrect resutSet.rows.length value: #{resutSet.rows.length} (expected: 1)"
                return

              if !resutSet.rows.item(0).TestColumn
                SelfTest.finishWithError errorcb,
                  'Missing resutSet.rows.item(0).TestColumn'
                return

              if resutSet.rows.item(0).TestColumn isnt 'test-value'
                SelfTest.finishWithError errorcb,
                  "Incorrect resutSet.rows.item(0).TestColumn value: #{resutSet.rows.item(0).TestColumn} (expected: 'test-value')"
                return

              db.transaction (tx) ->
                tx.executeSql 'UPDATE TestTable SET TestColumn = ?', ['new-value']
              , (tx_err) ->
                SelfTest.finishWithError errorcb, "UPDATE transaction error: #{tx_err}"
              , () ->
                db.readTransaction (tx2) ->
                  tx2.executeSql 'SELECT * FROM TestTable', [], (ignored, resutSet2) ->
                    if !resutSet2.rows
                      throw newSQLError 'Missing resutSet.rows'

                    if !resutSet2.rows.length
                      throw newSQLError 'Missing resutSet.rows.length'

                    if resutSet2.rows.length isnt 1
                      throw newSQLError "Incorrect resutSet.rows.length value: #{resutSet.rows.length} (expected: 1)"

                    if !resutSet2.rows.item(0).TestColumn
                      throw newSQLError 'Missing resutSet.rows.item(0).TestColumn'

                    if resutSet2.rows.item(0).TestColumn isnt 'new-value'
                      throw newSQLError "Incorrect resutSet.rows.item(0).TestColumn value: #{resutSet.rows.item(0).TestColumn} (expected: 'test-value')"

                , (tx2_err) ->
                  SelfTest.finishWithError errorcb, "readTransaction error: #{tx2_err}"
                , () ->
                  # CLEANUP & FINISH:
                  db.close () ->
                    SQLiteFactory.deleteDatabase {name: SelfTest.DBNAME, location: 'default'}, successcb, (cleanup_err)->
                      # TBD IGNORE THIS ERROR on Windows (and WP8):
                      if /Windows /.test(navigator.userAgent) or /IEMobile/.test(navigator.userAgent)
                        console.log "IGNORE CLEANUP (DELETE) ERROR: #{JSON.stringify cleanup_err} (Windows/WP8)"
                        successcb()
                        return
                      SelfTest.finishWithError errorcb, "Cleanup error: #{cleanup_err}"

                  , (close_err) ->
                    # TBD IGNORE THIS ERROR on Windows (and WP8):
                    if /Windows /.test(navigator.userAgent) or /IEMobile/.test(navigator.userAgent)
                      console.log "IGNORE close ERROR: #{JSON.stringify close_err} (Windows/WP8)"
                      SQLiteFactory.deleteDatabase {name: SelfTest.DBNAME, location: 'default'}, successcb, successcb
                      return
                    SelfTest.finishWithError errorcb, "close error: #{close_err}"

            , (select_err) ->
              SelfTest.finishWithError errorcb, "SELECT error: #{select_err}"

          , (batch_err) ->
            SelfTest.finishWithError errorcb, "sql batch error: #{batch_err}"

        , (open_err) ->
          SelfTest.finishWithError errorcb, "Open database error: #{open_err}"
        return

      finishWithError: (errorcb, message) ->
        console.log "selfTest ERROR with message: #{message}"
        SQLiteFactory.deleteDatabase {name: SelfTest.DBNAME, location: 'default'}, ->
          errorcb newSQLError message
          # FUTURE TODO: return
        # FUTURE TODO log err2
        , (err2)-> errorcb newSQLError "Cleanup error: #{err2} for error: #{message}"
        return

## Exported API:

    root.sqlitePlugin =
      sqliteFeatures:
        isSQLitePlugin: true

      echoTest: (okcb, errorcb) ->
        ok = (s) ->
          if s is 'test-string'
            okcb()
          else
            errorcb "Mismatch: got: '#{s}' expected 'test-string'"

        error = (e) ->
          errorcb e

        cordova.exec okcb, errorcb, "SQLitePlugin", "echoStringValue", [{value:'test-string'}]

      selfTest: SelfTest.start

      openDatabase: SQLiteFactory.openDatabase
      deleteDatabase: SQLiteFactory.deleteDatabase

## vim directives

#### vim: set filetype=coffee :
#### vim: set expandtab :
