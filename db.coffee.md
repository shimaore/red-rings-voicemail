Voicemail-database cache
========================

    fromJS = require './from-js'

    module.exports = (url) ->
      backend url, null, fromJS, true
