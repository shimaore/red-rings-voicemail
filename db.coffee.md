CouchDB backend cache
=====================

    LRU = require 'lru-cache'
    {EventEmitter2} = require 'eventemitter2'

    ev = new EventEmitter2 newListener: false, verboseMemoryLeak: true

    options =
      max: 200
      dispose: (key) -> ev.emit key
      maxAge: 20*60*1000

    cache = LRU options

    couchdb_backend = require 'red-rings-couchdb/backend'
    most = require 'most'

This is relatively generic, however it assumes the same view and reviver
are always used for the same URL (which is not true, generally speaking).

    backend = (url,view,reviver) ->
      if cache.has url
        cache.get url
      else
        db = couchdb_backend url, view, reviver
        cache.set url, db
        db.until most.fromEvent url, ev

Voicemail-database cache
========================

    fromJS = require './from-js'

    module.exports = (url) -> backend url, null, fromJS
