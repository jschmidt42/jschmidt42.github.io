---
layout: post
title: Convert JSON object to Lua (string)
category: code
tags: [json, javascript, lua, conversion]
---

Here's a snippet to convert a JSON object to a lua table/value.
```js
JSON.toLua = function (obj) {
    if (obj === null || obj === undefined) {
        return "nil";
    }
    if (!_.isObject(obj)) {
        if (typeof obj === 'string') {
            return '"' + obj + '"';
        }
        return obj.toString();
    }
    var result = "{";
    var isArray = obj instanceof Array;
    var len = _.size(obj);
    var i = 0;
    _.forEach(obj, function (v, k) {
        if (isArray) {
            result += JSON.toLua(v);
        } else {
            result += '["' + k + '"] = ' + JSON.toLua(v);
        }
        if (i < len-1) {
            result += ",";
        }
        ++i;
    });
    result += "}";
    return result;
};
```

Here's a few usage examples:

```js
JSON.toLua([42, null, [77, 45], "test", true, {a:66, "$b": 99}]);
```

```lua
{42,nil,{77,45},"test",true,{["a"] = 66,["$b"] = 99}}
```

```js
JSON.toLua([
  {"deltaX":23,"deltaY":21,"x":256,"y":722,"altKey":false,"ctrlKey":false,"shiftKey":false,"inputType":5}, 
  {"deltaX":19,"deltaY":15,"x":275,"y":707,"altKey":false,"ctrlKey":false,"shiftKey":false,"inputType":5}
]);
```

```lua
{
  {["deltaX"] = 23,["deltaY"] = 21,["x"] = 256,["y"] = 722,["altKey"] = false,["ctrlKey"] = false,["shiftKey"] = false,["inputType"] = 5},
  {["deltaX"] = 19,["deltaY"] = 15,["x"] = 275,["y"] = 707,["altKey"] = false,["ctrlKey"] = false,["shiftKey"] = false,["inputType"] = 5}
}
```