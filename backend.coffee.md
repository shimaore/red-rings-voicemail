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

      subscriptions = source
        .filter operation SUBSCRIBE
        .map (msg) ->
          key = msg.get 'key'

Subscribe to invidivual document

          if typeof key is 'string'
            $ = key.match VOICEMAIL_SETTINGS
            if $?
              return {msg,$}
          {msg}
        .filter ({$}) -> $?
        .map ({msg,$}) ->
          name = $[1]
          msg = msg
            .set 'key', 'voicemail_settings'
          {msg,name}

      most
      .mergeArray [ updates, subscriptions ]
      .map ({msg,name}) ->
        url = new URL "/#{ec name}", user_db_base
        {msg,url,name}
      .chain ({msg,url,name}) ->
        id_tr = (id) -> if id? and id is 'voicemail_settings' then "voicemail-settings:#{name}" else id
        the_db = couchdb_backend url.toString(), msg_view, fromJS
        the_db most.just msg
        .recoverWith (error) ->
          console.log 'Stream failed', error
          couchdb_backend url.toString(), null, fromJS
        .recoverWith (error) ->
          console.log 'Stream failed again', error
          most.empty()

The timer should probably be shorter and/or extensible (so that we can cache those instances in an LRU and reuse them).
Optimally we want to have a single instance running (here we might have dozens) — which, by the way, should also be shared with the one used by voicemail-messages, since we are monitoring changes _on the same database_.
So there's an abstraction layer missing here, some kind of LRU mechanism for the _database_ itself. (The stream should get closed when we get evicted, while the LRU should be updated every time we get a message for the database, whether it's UPDATE or SUBSCRIBE, and whether it regards voicemail settings, voicemail messages, or something else. Then this mechanims can be re-used for other types of databases, e.g. rates.)

        .until most.just(1).delay 10*60*1000

Inject placeholders for `music.mp3` and `ringback.mp3` (from huge-play/middleware/client/ingress/post), `prompt.mp3` and `name.mp3` (from well-groomed-feast/src/User).
Notice that huge-play uses `music.wav` and `ringback.wav` as default for backward compatibility, but we do not support uploading those unless they are already present (in other words setting `custom_music` and `custom_ringback` in local-numbers to `true`).

        .map (msg) ->
          msg = msg.updateIn ['doc','_attachments'], default_Map
          for file in [ 'music.mp3', 'ringback.mp3', 'prompt.mp3', 'name.mp3' ]
            msg = msg.updateIn ['doc','_attachments',file], default_Map
          msg
        .continueWith ->
          most.just Immutable.Map
            op: NOTIFY
            key: 'voicemail_settings'
            value: Immutable.Map reload: true
        .map (msg) ->
          msg
            .update 'key', id_tr
            .update 'id', id_tr
            .updateIn ['doc','_id'], id_tr

    default_Map = (x) -> x ? Immutable.Map {}

    module.exports = voicemail_settings
    couchdb_backend = require 'red-rings-couchdb/backend'
    ec = encodeURIComponent
    all_map = (view) -> (emit) -> view {emit}
    {app} = require 'well-groomed-feast-view'
    msg_view =
      map: all_map require 'well-groomed-feast-view/all'
      app: app
      name: 'all'
    most = require 'most'
    Immutable = require 'immutable'
    {operation} = require 'abrasive-ducks-transducers'
    {NOTIFY,UPDATE,SUBSCRIBE} = require 'red-rings/operations'
    {URL} = require 'url'
