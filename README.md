# lua-promises

[A+ promises](https://promisesaplus.com/) in Lua rewritten to [middleclass](https://github.com/kikito/middleclass).

## Why would I need promises?

Lua is normally single-threaded, so there is little need in asynchronous
operations. However, if you use HTTP requests, or sockets, or other types of
I/O - most likely your library would have asynchronous API:
	
``` lua
readfile('file.txt', function(contents, err)
	if err then
		print('Error', err)
	else
		-- process file contents
	end
end)
```

Using callbacks like this can quickly become problematic (once you need to make
multiple asyncrhonous actions depending on each other results). Let's imagine
some protocol where we need to connect to the remote peer, perform some
authentication, then send a request and finally receive some response:

``` lua
connect(function(status, err)
	if err then .... end
	auth(function(token, err)
		if err then ... end
		request(token, function(res, err)
			if err then ... end
			handleresult(res)
		end)
	end)
end)
```

Here's how the code could be rewritten using promises API:

``` lua
connect():next(function(status)
	return auth()
end):next(function(token)
	return request(token)
end):next(function(result)
	handleresult(res)
end, function(err)
	...handle error...
end)
```

This is cleaner and more readable, since it doesn't use lost of nested
callbacks. Also it has a single place to handle all errors.

The idea is that each function return an object which can be later resolved or
rejected. Various callbacks could be added to the object to get notified when
the object is resolved. Such objects are called promises, deferred objects,
thennables - all these names describe pretty much the same behavior.

## Requirements

* [middleclass](https://github.com/kikito/middleclass)

## Install

Download the `deferred.lua` from this repository.

Then in Lua code:
``` lua
local Deferred = require('deferred')
```

## API

Create new promises:

* `d = Deferred:new()` - returns a new promise object `d`
* `d = Deferred:all(promises)` - returns a new promise object `d` that is
	resolved when all `promises` are resolved/rejected.
* `d = Deferred:first(promises)` - returns a new promise object `d` that is
	resolved as soon as the first of the `promises` gets resolved/rejected.
* `d = Deferred:map(list, fn)` - returns a new promise object `d` that is
	resolved with the values of sequential application of function `fn` to each
	element in the `list`. `fn` is expected to return promise object.

Resolve/reject:

* `d:resolve(value)` - resolve promise object with `value`
* `d:reject(value)` - reject promise object with `value`

Wait for the promise object:

* `d:next(cb, [errcb])` - enqueues resolve callback `cb` and (optionally) a
	rejection callback `errcb`. Resolve callback can be nil.

## Example

``` lua
local Deferred = require('deferred')

--
-- Converting callback-based API into promise-based is very straightforward:
-- 
-- 1) Create promise object
-- 2) Start your asynchronous action
-- 3) Resolve promise object whenever action is finished (only first resolution
--    is accepted, others are ignored)
-- 4) Reject promise object whenever action is failed (only first rejection is
--    accepted, others are ignored)
-- 5) Return promise object letting calling side to add a chain of callbacks to
--    your asynchronous function

function read(f)
	local d = Deferred:new()
	readasync(f, function(contents, err)
		if err == nil then
			d:resolve(contents)
		else
			d:reject(err)
		end
	end)
	return d
end

-- You can now use read() like this:
read('file.txt'):next(function(s)
	print('File.txt contents: ', s)
end, function(err)
	print('Error', err)
end)
```

## Chaining promises

Promises can be chained (read A+ specs for more details). It's convenient when
you need to do several asynchronous actions sequentially. Each callback can
return another promise object, then further callbacks could wait for it to
become resolved/rejected:

``` lua
-- Reading two files sequentially:
read('first.txt'):next(function(s)
	print('File file:', s)
	return read('second.txt')
end):next(function(s)
	print('Second file:', s)
end):next(nil, function(err)
	-- error while reading first or second file
	print('Error', err)
end)
```

## Processing lists

You can process a list of object asynchronously, so the next asynchronous
action is started only when the previous one is successfully completed:

``` lua
local items = {'a.txt', 'b.txt', 'c.txt'}
-- Read 3 files, one by one
Deferred:map(items, read):next(function(files)
	-- here files is an array of file contents for each of the files
end, function(err)
	-- handle reading error
end)
```

## Waiting for a group of promises

You may start multiple asynchronous actions in parallel and wait for all of
them to complete:

``` lua
Deferred:all({
	http.get('http://example.com/first'),
	http.get('http://example.com/second'),
	http.get('http://example.com/third'),
}):next(function(results)
	-- handle results here (all requests are finished and there has been
	-- no errors)
end, function(results)
	-- handle errors here (all requests are finished and there has been
	-- at least one error)
end)
```

## Waiting for the first promise

In some cases it's handy to wait for either of the promises. A good example is reading with timeout:

``` lua

-- returns a promise that gets rejected after a certain timeout
function timeout(sec)
	local d = Deferred:new()
	settimeout(function()
		d:reject('Timeout')
	end, sec)
	return d
end

Deferred:first({
	read(somefile), -- resolves promise with contents, or rejects with error
	timeout(5),
}):next(function(result)
	...file was read successfully...
end, function(err)
	...either timeout or I/O error...
end)
```

## License

Code is distributed under MIT license.
