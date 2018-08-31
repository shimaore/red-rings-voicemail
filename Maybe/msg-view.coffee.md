If `well-groomed-feast-view` ever gets deployed we may use this code, in
conjunction with `backend url, msg_view, fromJS` in `db.coffee.md`.

    all_map = (view) -> (emit) -> view {emit}
    {app} = require 'well-groomed-feast-view'
    msg_view =
      map: all_map require 'well-groomed-feast-view/all'
      app: app
      name: 'all'
    module.exports = msg_view
