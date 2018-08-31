    reviver = require 'ccnq4-kitty/reviver'
    Immutable = require 'immutable'
    fromJS = (x) -> Immutable.fromJS x, reviver

    module.exports = fromJS
