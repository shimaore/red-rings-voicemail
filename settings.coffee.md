CouchDB backend to access the voicemail-settings records
========================================================

    VOICEMAIL_SETTINGS = /^voicemail-settings:(u[\w-]+)$/

    voicemail_settings = (source,user_db_base,fromJS) ->

      updates = source
        .filter operation UPDATE
        .map (msg) -> {msg,$:msg.get('id')?.match? VOICEMAIL_SETTINGS}
        .filter ({$}) -> $?
        .map ({msg,$}) ->
          name = $[1]
          msg = msg
            .set 'id', 'voicemail_settings'
            .setIn ['doc','_id'], 'voicemail_settings'
          {msg,name}

One key format is handled:
- `voicemail-settings:<db>` to retrieve the settings

      subscriptions = source
        .filter operation SUBSCRIBE
        .map (msg) ->
          key = msg.get 'key'

Subscribe to invidivual document

          if typeof key is 'string'
            $ = key.match VOICEMAIL_SETTINGS
            if $?
              msg = msg.set 'key', 'voicemail_settings'
              name = $[1]
              return {msg,name}

No match (will get filtered out)

          {}
        .filter ({name}) -> name?

      most
      .mergeArray [ updates, subscriptions ]
      .map ({msg,name}) ->
        url = new URL "/#{ec name}", user_db_base
        {msg,url,name}
      .chain ({msg,url,name}) ->

Translate key/id back to the request format (see last `map` down).

        tr = (v) ->
          switch
            when v? and v is 'voicemail_settings'
              "voicemail-settings:#{name}"
            else
              console.error "Unexpected id/key in response to #{name}", v
              v

        id_tr = key_tr = tr

Send to the backend processor.

        the_db = couchdb_backend url.toString()
        the_db most.just msg

On the return stream (i.e. in NOTIFY events sent in response to SUBSCRIBE), inject placeholders for `music.mp3` and `ringback.mp3` (from huge-play/middleware/client/ingress/post), `prompt.mp3` and `name.mp3` (from well-groomed-feast/src/User).
Notice that huge-play uses `music.wav` and `ringback.wav` as default for backward compatibility, but we do not support uploading those unless they are already present (in other words setting `custom_music` and `custom_ringback` in local-numbers to `true`).

        .map (msg) ->
          msg = msg.updateIn ['doc','_attachments'], default_Map
          for file in [ 'music.mp3', 'ringback.mp3', 'prompt.mp3', 'name.mp3' ]
            msg = msg.updateIn ['doc','_attachments',file], default_Map
          msg

Log stream errror, allow to send reload notification

        .recoverWith (error) ->
          console.error 'Stream failed', error
          most.empty()

Notification at end of stream

        .continueWith ->
          console.error 'Sending NOTIFY reload: true'
          most.just Immutable.Map
            op: NOTIFY
            key: 'voicemail_settings'
            value: Immutable.Map reload: true

Re-inject any missing field

        .map (msg) ->
          msg
            .update 'key', key_tr
            .update 'id', id_tr
            .updateIn ['doc','_id'], id_tr

    default_Map = (x) -> x ? Immutable.Map {}
    ec = encodeURIComponent

    module.exports = voicemail_settings
    couchdb_backend = require './db'
    most = require 'most'
    Immutable = require 'immutable'
    {operation} = require 'abrasive-ducks-transducers'
    {NOTIFY,UPDATE,SUBSCRIBE} = require 'red-rings/operations'
    {URL} = require 'url'
