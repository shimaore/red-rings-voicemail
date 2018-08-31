Voicemail-database cache
========================

    couchdb_backend = require 'red-rings-couchdb/backend'
    fromJS = require './from-js'

    module.exports = (url) ->
      couchdb_backend url, null, fromJS, true
