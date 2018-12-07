CouchDB backend to access the voicemail messages
================================================

    VOICEMAIL_MESSAGE  = /^voicemail:(u[\w-]+)(?::(\S+))?$/

    voicemail_messages = (source,user_db_base,fromJS) ->

      updates = source
        .filter operation UPDATE
        .map (msg) -> {msg,$:msg.get('id')?.match? VOICEMAIL_MESSAGE}
        .filter ({$}) -> $? and $[2]?
        .map ({msg,$}) ->
          name = $[1]
          msg = msg
            .set 'id', "voicemail:#{$[2]}"
            .setIn ['doc','_id'], "voicemail:#{$[2]}"
          {msg,name}

Two key formats are handled:
- `voicemail:<db>` to retrieve the list of messages
- `voicemail:<db>:<id>` to retrieve a specific message

      subscriptions = source
        .filter operation SUBSCRIBE
        .map (msg) ->
          key = msg.get 'key'

          if typeof key is 'string'
            $ = key.match VOICEMAIL_MESSAGE
            if $?

Subscribe to invidivual document

              if $[2]
                msg = msg.set 'key', "voicemail:#{$[2]}"
                return {msg,name:$[1]}

Gather list of messages, using the min/max view integrated in red-rings-couchdb.

              else
                msg = msg.set 'key', Immutable.Map min:'voicemail:', max:'voicemail;'
                return {msg,name:$[1]}

No match (will get filtered out)

          {}
        .filter ({name}) -> name?

      most
      .mergeArray [ updates, subscriptions ]
      .map ({msg,name}) ->
        url = new URL "/#{ec name}", user_db_base
        {msg,url,name}
      .chain ({msg,url,name}) ->

        id_tr = (id) ->
          switch
            # document
            when $ = id?.match /^voicemail:(.+)$/
              "voicemail:#{name}:#{$[1]}"

            else
              debug "Unexpected id in response to #{name}", id
              id

        key_tr = (key) ->
          switch
            # single document
            when 'string' is typeof key
              "voicemail:#{name}#{$[1]}"
            # min/max view
            when Immutable.Map.isMap(key) and key.has 'min'
              "voicemail:#{name}"

            else
              debug "Unexpected key in response to #{name}", key
              key

        the_db = couchdb_backend url.toString()
        the_db most.just msg

Re-inject any missing field

        .map (msg) ->
          msg
            .update 'key', key_tr
            .update 'id', id_tr
            .updateIn ['doc','_id'], id_tr

    module.exports = voicemail_messages
    couchdb_backend = require './db'
    most = require 'most'
    ec = encodeURIComponent
    Immutable = require 'immutable'
    {operation} = require 'abrasive-ducks-transducers'
    {NOTIFY,UPDATE,SUBSCRIBE} = require 'red-rings/operations'
    {debug} = (require 'tangible') 'red-rings-voicemail:messages'
    {URL} = require 'url'
