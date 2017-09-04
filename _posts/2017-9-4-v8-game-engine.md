---
layout: post
title: V8 integration in a game engine
tags: [v8, game, engine, javascript, debugger]
---

First thing first, please read the [V8 Embedder's Guide](https://developers.google.com/v8/embed) to understand V8 base concepts. The following is a way to integrate V8 in a game engine.

# Motivation

Recently I've been doing a lot of JavaScript for various projects and JavaScript has become one of my favorite languages. Since I am maintaining a small game engine for the past few years, I've told myself; why not use JavaScript to code the game logic of my game using Engine42. Engine42 is a small game engine coded in C++ that has a D3D9 renderer and uses an entity-components system.

The idea is to have a script engine that the game engine uses to operate the game logic. The engine uses the script engine to load and evaluate JavaScript code. The script engine provides 4 main entry points in which the user can do about anything. Also, the game engine is responsible to expose core  systems to the scripting language and basic JavaScript concepts such as `console`, `setTimeout`, `clearTimeout`, `require`, etc. 

Minimally a main script looks like this:

```js
function init() {
    "use strict";

	console.log('Called when the system is initialized');
	console.warn('TODO: Create you global and main game objects here.');
}

function update(dt) {
    "use strict";

	console.log('Called when the engine is processing a tick.')
	console.info('You can use', dt, 
        'to know how much time has elapsed since the last call to update().');

	console.warn('TODO: Updates your game state.').

	console.info('Return', true, 'if you want to continue, otherwise return', false, 
        'if you want the game to exit.');
	return true;
}

function render() {
    "use strict";
	
	console.log('The engine is ready to render elements.');
	console.info('Call for each frames, usally after update().');
	
	// Render all entities
	var entitySystem = require('entity'); // More on require('...') later
	entitySystem.root.render();
}

function shutdown() {
    "use strict";
	
	console.log('Called when the system is about to shutdown.');
	console.warn('TODO: Do some cleanup.');
}
```

The 4 functions defined above will be called by the engine at specific times. These are the only functions the engine will invoke explicitly. Everything else executed by the JavaScript VM will be called from the following functions and boot script.

It is important to note that the boot script is evaluated before calling `init()`. So in the global scope, the user can require other modules, run some JS code and initialize global variables. It is fine to require system modules from the global closure, but it is recommended to initialize and create game objects in the `init()` closure.

If one of the following functions is not defined at start-up, the engine asserts and quit.

# Initialization

Before running any scripts, we need to initialize V8 and register a few systems that will be used by the scripts.

```cpp
void ScriptEngine::InitializeV8()
{
	// Initialize V8.
	V8::InitializeICU();
	mPlatform = platform::CreateDefaultPlatform();
	V8::InitializePlatform(mPlatform);
	V8::Initialize();

#if !defined(FINAL)
	const char* v8Flags = "--debugger --expose_debug_as=v8debug";
	V8::SetFlagsFromString(v8Flags, strlen(v8Flags));
#endif

	// Define a custom allocator for V8 to allocate some memory when needed.
	v8::V8::SetArrayBufferAllocator(new MallocArrayBufferAllocator);

	// Create a new Isolate and make it the current one.
	mIsolate = Isolate::New();
	mIsolate->Enter();

	// Create a stack-allocated handle scope.
	HandleScope scope(mIsolate);

	// Create a new context.
	Local<v8::Context> context = Context::New(mIsolate);
	mGlobalContext.Reset(mIsolate, context);
	Context::Scope contextScope(context);

	// Create local to self
	Handle<ObjectTemplate> t = ObjectTemplate::New();
	t->SetInternalFieldCount(1);
	mSelf = t->NewInstance();
	mSelf->SetInternalField(0, External::New(mIsolate, this));

#if !defined(FINAL)
	// Enable the debugger thread to support remote debugging.
	v8::Debug::SetMessageHandler(DebuggerAgentMessageHandler);
	v8::Debug::SetDebugEventListener(HandleDebugEventCallback, mSelf);
	mDebuggerThread.EnqueueFunctionCall(FunctionCall(&ScriptEngine::DebuggerThread, *this));
#endif

	// Expose main systems to the scripts
	Console::Create(mIsolate, context);
	ScriptWorldCore::Create(mIsolate, context);
	ScriptEntityCore::Create(mIsolate, context);
	ScriptRenderCore::Create(mIsolate, context);
	ScriptVector3::Create(mIsolate, context);
	ScriptQuaternion::Create(mIsolate, context);
	ScriptEntity::Create(mIsolate, context);
	ScriptComponent::Create(mIsolate, context);

	// Defines common JavaScript global functions
	context->Global()->Set(String::NewFromUtf8(mIsolate, "require"), 
        FunctionTemplate::New(mIsolate, Require, mSelf)->GetFunction());
	context->Global()->Set(String::NewFromUtf8(mIsolate, "setTimeout"), 
        FunctionTemplate::New(mIsolate, SetTimeout, mSelf)->GetFunction());
	context->Global()->Set(String::NewFromUtf8(mIsolate, "clearTimeout"), 
        FunctionTemplate::New(mIsolate, ClearTimeout, mSelf)->GetFunction());
}
```

Lets describe each sections.

## V8 initialization

```cpp
V8::InitializeICU();
mPlatform = platform::CreateDefaultPlatform();
V8::InitializePlatform(mPlatform);
V8::Initialize();
```

These are the four main lines that initialize V8. These are the first call you need to make before doing anything with V8.

## Enable V8 debugging

```cpp
#if !defined(FINAL)
	const char* v8Flags = "--debugger --expose_debug_as=v8debug";
	V8::SetFlagsFromString(v8Flags, strlen(v8Flags));
#endif
```

When the engine is compiled in development mode (Debug or Release) we make sure V8 enables debugging support. The `--expose_debug_as=v8debug` injects in the global context a variable named `v8debug` that can be used by the remote debugger to test some states. For instance Webstorm uses that variables to check the debugger's state.

## V8 allocators

In order to allow V8 to allocate some memory, you need to define your own allocators.

```cpp
// Define a custom allocator for V8 to allocate some memory when needed.
v8::V8::SetArrayBufferAllocator(new MallocArrayBufferAllocator);
```

`MallocArrayBufferAllocator` is defined by us:

```cpp
class MallocArrayBufferAllocator : public v8::ArrayBuffer::Allocator {
public:
	virtual void* Allocate(size_t length) { return malloc(length); }
	virtual void* AllocateUninitialized(size_t length) { return malloc(length); }
	virtual void Free(void* data, size_t length) { free(data); }
};
```

For the example we use a simple allocator, but you can think of something more complex and implement something to track V8 memory usage. You may not need an allocator if you only execute simple scripts, but as soon as you compile and evaluate bigger scripts, V8 will require one.

In case you didn't register an allocator, V8 will crash with an error such as 

```
#
# Fatal error in ..\..\src\runtime.cc, line 801
# CHECK(V8::ArrayBufferAllocator() != NULL) failed
#
First-chance exception at 0x00000000 in GameApp.exe: 0xC0000005: Access violation executing location 0x00000000.
Unhandled exception at 0x77C70309 in GameApp.exe: 0xC0000005: Access violation executing location 0x00000000.
```

## V8 Isolation

The next step is to create an isolated V8 engine. In example, if your game engine has sandboxes, then each sandbox could have their own V8 isolate context. Here we have a single isolate context for the entire engine. Everything you do with V8 needs an isolated context so keep that member handy.

```cpp
// Create a new Isolate and make it the current one.
mIsolate = Isolate::New();
mIsolate->Enter();
```

Here we enter the isolation context right away and we only exit the isolation context when we shutdown the engine. There is no need to better scope the isolation context as we'll need V8 always ready for all steps, `init`, `update`, `render` to `shutdown`.

## Handle scope

As soon as you need to create some handles, you need to create a handle scope so V8 can properly manage allocated variables.

```cpp
// Create a stack-allocated handle scope.
HandleScope scope(mIsolate);
```

In case you forgot to properly manage your scopes, V8 will give you a little reminder:


```
#
# Fatal error in D:\node\deps\v8\src/handles-inl.h, line 126
# CHECK(current->level > 0) failed
#
First-chance exception at 0x00000000 in GameApp.exe: 0xC0000005: Access violation executing location 0x00000000.
Unhandled exception at 0x77C70309 in GameApp.exe: 0xC0000005: Access violation executing location 0x00000000.
```

If you look at the callstack, you'll see that V8 is trying to get the current scope:

```
>	GameApp.exe!v8::base::OS::Abort() Line 831	C++
 	GameApp.exe!V8_Fatal(const char * file, int line, const char * format, ...) Line 88	C++
 	GameApp.exe!v8::internal::HandleScope::CloseAndEscape<v8::internal::Context>(v8::internal::Handle<v8::internal::Context> handle_value) Line 126	C++
 	GameApp.exe!v8::Context::New(v8::Isolate * external_isolate, v8::ExtensionConfiguration * extensions, v8::Handle<v8::ObjectTemplate> global_template, v8::Handle<v8::Value> global_object) Line 5170	C++
 	GameApp.exe!ScriptEngine::InitializeV8() Line 164	C++
```

So when you get these, just add `HandleScope scope(mIsolate);` and you should be set.

## V8 context

When you are to evaluate JavaScript code, you need to define a context. In our case we create a global context that we persist for the entire session:

```cpp
	// Create a new context.
	Local<v8::Context> context = Context::New(mIsolate);
	mGlobalContext.Reset(mIsolate, context);
	Context::Scope contextScope(context);
```

`mGlobalContext` is a persistent context defined as `v8::Persistent<v8::Context>`.

A persistent template is an object reference that is independent of any handle scope. That means that you can store and use them anywhere (almost).

Persistent objects are a bit tricky to access when needed as locals (e.g. `Local<Value>`). To do so, I have create some helper functions.

```cpp
	template <class TypeName>
	inline v8::Local<TypeName> StrongPersistentToLocal(
        const v8::Persistent<TypeName>& persistent)
	{
		return *reinterpret_cast<v8::Local<TypeName>*>(
            const_cast<v8::Persistent<TypeName>*>(&persistent));
	}

	template <class TypeName>
	inline v8::Local<TypeName> StrongPersistentToLocal(
        const v8::Persistent<TypeName>& persistent) const
	{
		return *reinterpret_cast<const v8::Local<TypeName>*>(
            const_cast<const v8::Persistent<TypeName>*>(&persistent));
	}
```

You call `StrongPersistentToLocal` when you need to access persistent objects. In example, we use that function whenever we need to get the global context:

```cpp
	v8::Local<v8::Context> Context() { return StrongPersistentToLocal(mGlobalContext); }
	v8::Local<v8::Context> Context() const { return StrongPersistentToLocal(mGlobalContext); }

	// usage:
	void ScriptEngine::Boot()
	{
		HandleScope handle_scope(mIsolate);
		v8::Context::Scope contextScope(Context());
		///...
	}
```

## Expose script engine to V8

In the following snippet we expose the `ScriptEngine` singleton to V8. `Handle<ObjectTemplate> t = ObjectTemplate::New()` creates an new type of object that can be instantiated. `t->SetInternalFieldCount(1)` tells V8 that there is gonna be one internal value to this object. This value can't be used in the JavaScript code. Then we create an instance of that object `mSelf = t->NewInstance()` and we store the pointer `this` to that created object. We will retrieve that value later in the implementation of global functions such as `setTimeout` and `require`.

```cpp
	// Create local to self
	Handle<ObjectTemplate> t = ObjectTemplate::New();
	t->SetInternalFieldCount(1);
	mSelf = t->NewInstance();
	mSelf->SetInternalField(0, External::New(mIsolate, this));
```

## Starts the debugging socket server

Here we start the debugging socket server to listen to V8 debugging commands. The socket server is created and ran in another thread. 

```cpp
	// Enable the debugger thread to support remote debugging.
	v8::Debug::SetMessageHandler(DebuggerAgentMessageHandler);
	v8::Debug::SetDebugEventListener(HandleDebugEventCallback, mSelf);
	mDebuggerThread.EnqueueFunctionCall(FunctionCall(&ScriptEngine::DebuggerThread, *this));
```

Later we'll see how the debugger thread is implemented.

## Expose API

The next section expose to the JavaScript code all the APIs we want.

```cpp
	// Expose main systems to the scripts
	Console::Create(mIsolate, context);
	ScriptWorldCore::Create(mIsolate, context);
	ScriptEntityCore::Create(mIsolate, context);
	ScriptRenderCore::Create(mIsolate, context);
	ScriptVector3::Create(mIsolate, context);
	ScriptQuaternion::Create(mIsolate, context);
	ScriptEntity::Create(mIsolate, context);
	ScriptComponent::Create(mIsolate, context);
	...
```

`Console::Create(mIsolate, context)` creates the `console` object in the global context. 

We expose three kinds of APIs to the scripts:

- Global objects; objects that are already instantiated by the script engine.
	- console
	- Math, extends the Math namespace with vector, quaternion and matrix operation.
- Constructors; class that can be instantiated in the JavaScript.
	- Vectors
	- Matrices
	- Entity
	- Components
	- etc.
- Core systems (exposed when using `require`, see below)
	- Entity Core
	- Renderer Core
	- Resource manager
	- World Manager
	- etc.

We will see below how these are constructed and registered.

## Register common global functions

Finally in the initialization method we register common global function used in JavaScript such as `setTimeout`, `clearTimeout`, `require`, etc.

```cpp
	// Defines common JavaScript global functions
	context->Global()->Set(String::NewFromUtf8(mIsolate, "require"), 
        FunctionTemplate::New(mIsolate, Require, mSelf)->GetFunction());
	context->Global()->Set(String::NewFromUtf8(mIsolate, "setTimeout"), 
        FunctionTemplate::New(mIsolate, SetTimeout, mSelf)->GetFunction());
	context->Global()->Set(String::NewFromUtf8(mIsolate, "clearTimeout"), 
        FunctionTemplate::New(mIsolate, ClearTimeout, mSelf)->GetFunction());
```

We create function template to instantiate the functions. Once created the functions are added to the global context so these are available from any scripts.

# API - Global objects

Some objects are set on the global context directly such as the `console`. To do so we create a new object template and add function members to it. When all members are set, we push that object to the global context and give it a name.

```cpp
void Console::Create(Isolate* isolate, Local<Context> context)
{
	Handle<ObjectTemplate> console = ObjectTemplate::New(isolate);
	console->Set(String::NewFromUtf8(isolate, "log"), FunctionTemplate::New(isolate, Print));
	...
	context->Global()->Set(String::NewFromUtf8(isolate, "console"), console->NewInstance());
}
```

For each function, when called from the JavaScript code, V8 calls our C++ code in order to do something. In example, for the `log` method of the `console` object, V8 calls our Print callback.

```cpp
void Console::Print(const FunctionCallbackInfo<Value>& args)
{
	StringBuilder sb;
	for (int i = 0; i < args.Length(); i++)
	{
		if (i > 0)
			sb << " ";
		String::Utf8Value str(args[i]);
		sb << *str;
	}

	VCNLog << "[Script Console] " << sb.str() << std::endl;

	args.GetReturnValue().Set(args.Holder());
}
```

All the callbacks our static function.

# API - Global functions

Global functions are set on the global context just like the `console` object.

```cpp
void ScriptEngine::SetTimeout(const FunctionCallbackInfo<Value>& args)
{
	VCN_ASSERT(args.Length() >= 2);
	VCN_ASSERT(!args[0].As<Function>().IsEmpty());
	VCN_ASSERT(!args[1].As<Number>().IsEmpty());

	HandleScope scope(args.GetIsolate());
	ScriptEngine* scriptInterface = GetDataScriptInterface(args);

	int timeoutId = scriptInterface->mNextTimeoutId++;

	Local<Function> callback = args[0].As<Function>();
	double timeoutMs = args[1].As<Number>()->NumberValue();

	scriptInterface->mTimeouts[timeoutId] = Timeout(
		timeoutId,
		VCNSystem::GetInstance()->GetTotalElapsed() + (timeoutMs / 1000.0),
		GlobalFunction(args.GetIsolate(), callback));

	args.GetReturnValue().Set(Number::New(args.GetIsolate(), timeoutId));
}
```

These are still static function, but the difference with the Print function above is that here we have access to the script engine instance. Do you remember when we've defined the `self` object?

```cpp
	Handle<ObjectTemplate> t = ObjectTemplate::New();
	t->SetInternalFieldCount(1);
	mSelf = t->NewInstance();
	mSelf->SetInternalField(0, External::New(mIsolate, this));
```

That was to pass this data object to the registration of the global functions.

```cpp
	context->Global()->Set(String::NewFromUtf8(mIsolate, "setTimeout"), 
        FunctionTemplate::New(mIsolate, SetTimeout, mSelf)->GetFunction());
```

So when we've passed the `mSelf` object when registering `setTimeout`, V8 kept that object so it can pass it back to the callback. That is why in the `SetTimeout` callback code have `ScriptEngine* scriptInterface = GetDataScriptInterface(args)`.

The `GetDataScriptInterface` function extract the `this` from the function arguments.

```cpp
ScriptEngine* ScriptEngine::GetDataScriptInterface(const FunctionCallbackInfo<Value>& args)
{
	return GetDataScriptInterface(args.Data());
}

ScriptEngine* ScriptEngine::GetDataScriptInterface(Local<Value> data)
{
	Local<Object> self = data.As<Object>();
	Local<External> wrap = Local<External>::Cast(self->GetInternalField(0));
	return static_cast<ScriptEngine*>(wrap->Value());
}
```

Global function can be used in any scripts at anytime without using `require`. Here's a small snippet that uses `setTimeout`.

```js
	if (setTimeout) {
		console.log("setTimeout exists");

		setTimeout(function () {
			console.log('Called after 2 seconds');
		}, 2000);
	}
```

# API - Constructors Wrappers

Constructors is a way to expose to JavaScript classes that can be created on the JS side. In example we want to be able to do this in JavaScript:

```js
function initializeCameraPosition(camera) {
	var cameraPosition = new Vector3(22, 42, 24.5);
	camera.position = cameraPosition;
	camera.position.x += 2;

	camera.direction = new Vector3(1, 0, 0);
	camera.direction.y = 8.42f;
	camera.normalize();
}
```

In the example above we need two constructors API. One for Vector3 and Camera. All these constructors could be entirely coded in JavaScript using normal constructs, but since Camera and Vector are objects that are also used by the C++ engine, we need to have them declared in the C++ as V8 wrappers.

Here's how this is done for a simple Vector3. You'll see how fields and methods are defined. But first, to ease the wrapping of C++ classes, we have a `ScriptObjectWrapper` that is used as a base class.

```cpp
class ScriptObjectWrap
{
public:
	ScriptObjectWrap()
	{
		_refs = 0;
	}


	virtual ~ScriptObjectWrap()
	{
		if (persistent().IsEmpty())
			return;
		VCN_ASSERT(persistent().IsNearDeath());
		persistent().ClearWeak();
		persistent().Reset();
	}


	template <class T>
	static inline T* Unwrap(v8::Handle<v8::Object> handle)
	{
		VCN_ASSERT(!handle.IsEmpty());
		VCN_ASSERT(handle->InternalFieldCount() > 0);

		// Cast to ObjectWrap before casting to T.  A direct cast from void
		// to T won't work right when T has more than one base class.
		void* ptr = handle->GetAlignedPointerFromInternalField(0);
		ScriptObjectWrap* wrap = static_cast<ScriptObjectWrap*>(ptr);
		return static_cast<T*>(wrap);
	}

	inline v8::Local<v8::Object> handle()
	{
		return handle(v8::Isolate::GetCurrent());
	}

	inline v8::Local<v8::Object> handle(v8::Isolate* isolate)
	{
		return v8::Local<v8::Object>::New(isolate, persistent());
	}

	inline v8::Persistent<v8::Object>& persistent()
	{
		return _handle;
	}

protected:
	inline void Wrap(v8::Handle<v8::Object> handle)
	{
		VCN_ASSERT(persistent().IsEmpty());
		VCN_ASSERT(handle->InternalFieldCount() > 0);
		handle->SetAlignedPointerInInternalField(0, this);
		persistent().Reset(v8::Isolate::GetCurrent(), handle);
		MakeWeak();
	}

	inline void MakeWeak(void)
	{
		persistent().SetWeak(this, WeakCallback);
		persistent().MarkIndependent();
	}

	/* Ref() marks the object as being attached to an event loop.
	* Refed objects will not be garbage collected, even if
	* all references are lost.
	*/
	virtual void Ref()
	{
		assert(!persistent().IsEmpty());
		persistent().ClearWeak();
		_refs++;
	}

	/* Unref() marks an object as detached from the event loop.  This is its
	* default state.  When an object with a "weak" reference changes from
	* attached to detached state it will be freed. Be careful not to access
	* the object after making this call as it might be gone!
	* (A "weak reference" means an object that only has a
	* persistent handle.)
	*
	* DO NOT CALL THIS FROM DESTRUCTOR
	*/
	virtual void Unref()
	{
		assert(!persistent().IsEmpty());
		assert(!persistent().IsWeak());
		assert(_refs > 0);
		if (--_refs == 0)
			MakeWeak();
	}

	int _refs;

private:
	static void WeakCallback(const v8::WeakCallbackData<v8::Object, ScriptObjectWrap>& data)
	{
		v8::Isolate* isolate = data.GetIsolate();
		v8::HandleScope scope(isolate);
		ScriptObjectWrap* wrap = data.GetParameter();
		VCN_ASSERT(wrap->_refs == 0);
		VCN_ASSERT(wrap->_handle.IsNearDeath());
		VCN_ASSERT(data.GetValue() == v8::Local<v8::Object>::New(isolate, wrap->_handle));
		wrap->_handle.Reset();
		delete wrap;
	}

	v8::Persistent<v8::Object> _handle;
};
```

Then `ScriptVector3` is defined as:

```cpp
class ScriptVector3 : public ScriptObjectWrap
{
public:

	/// Called to register the Vector3 constructor in the global namespace
	static void Create(Isolate* isolate, Local<Context> context)
	{
		// Prepare constructor template
		Local<FunctionTemplate> tpl = FunctionTemplate::New(isolate, New);
		tpl->SetClassName(String::NewFromUtf8(isolate, "Vector3"));
		tpl->InstanceTemplate()->SetInternalFieldCount(1);

		// Prototype
		tpl->PrototypeTemplate()->Set(String::NewFromUtf8(isolate, "normalize"), 
           FunctionTemplate::New(isolate, Normalize)->GetFunction());

		// Properties
		tpl->InstanceTemplate()->SetAccessor(String::NewFromUtf8(isolate, "x"), GetX, SetX);
		tpl->InstanceTemplate()->SetAccessor(String::NewFromUtf8(isolate, "y"), GetY, SetY);
		tpl->InstanceTemplate()->SetAccessor(String::NewFromUtf8(isolate, "z"), GetZ, SetZ);

		// Define the constructor that will be used in JS to create a Vector3
		_constructor.Reset(isolate, tpl->GetFunction());

		context->Global()->Set(String::NewFromUtf8(isolate, "Vector3"), 
           StrongPersistentToLocal(_constructor));
	}

	/// Called in C++ to create a new instance that can be passed to JS.
	static v8::Handle<v8::Value> NewInstance(Isolate* isolate, float x, float y, float z)
	{
		HandleScope scope(isolate);

		Handle<Function> constructor = StrongPersistentToLocal(_constructor);

		Local<Object> instance = constructor->NewInstance();
		ScriptVector3* obj = ScriptObjectWrap::Unwrap<ScriptVector3>(instance);
		obj->_vec = Vector3(x, y, z);

		return instance;
	}

private:
	ScriptVector3(float x, float y, float z)
		: _vec(x, y, z)
	{
	}

	~ScriptVector3()
	{
	}

	// Constructor
	static void New(const FunctionCallbackInfo<Value>& info)
	{
		HandleScope scope(info.GetIsolate());

		VCN_ASSERT_MSG(info.Length() == 3, "Bad arguments");
		VCN_ASSERT_MSG(info[0]->IsNumber(), "x must be a number");
		VCN_ASSERT_MSG(info[1]->IsNumber(), "y must be a number");
		VCN_ASSERT_MSG(info[2]->IsNumber(), "z must be a number");

		float x = (float)info[0]->ToNumber()->Value();
		float y = (float)info[0]->ToNumber()->Value();
		float z = (float)info[0]->ToNumber()->Value();

		ScriptVector3* obj = new ScriptVector3(x, y, z);
		obj->Wrap(info.This());

		return info.GetReturnValue().Set(info.This());
	}

	// Accessors
	static void GetX(Local<String> property, const PropertyCallbackInfo<Value>& info)
	{
		const ScriptVector3* obj = ScriptObjectWrap::Unwrap<ScriptVector3>(info.This());
		info.GetReturnValue().Set(v8::Number::New(info.GetIsolate(), obj->_vec.x));
	}
	static void GetY(Local<String> property, const PropertyCallbackInfo<Value>& info)
	{
		const ScriptVector3* obj = ScriptObjectWrap::Unwrap<ScriptVector3>(info.This());
		info.GetReturnValue().Set(v8::Number::New(info.GetIsolate(), obj->_vec.y));
	}
	static void GetZ(Local<String> property, const PropertyCallbackInfo<Value>& info)
	{
		const ScriptVector3* obj = ScriptObjectWrap::Unwrap<ScriptVector3>(info.This());
		info.GetReturnValue().Set(v8::Number::New(info.GetIsolate(), obj->_vec.z));
	}

	static void SetX(Local<String> property, Local<Value> value, 
        const PropertyCallbackInfo<void>& info)
	{
		ScriptVector3* obj = ScriptObjectWrap::Unwrap<ScriptVector3>(info.This());
		obj->_vec.x = (float)value->ToNumber()->Value();
	}
	static void SetY(Local<String> property, Local<Value> value, 
        const PropertyCallbackInfo<void>& info)
	{
		ScriptVector3* obj = ScriptObjectWrap::Unwrap<ScriptVector3>(info.This());
		obj->_vec.y = (float)value->ToNumber()->Value();
	}
	static void SetZ(Local<String> property, Local<Value> value, 
        const PropertyCallbackInfo<void>& info)
	{
		ScriptVector3* obj = ScriptObjectWrap::Unwrap<ScriptVector3>(info.This());
		obj->_vec.z = (float)value->ToNumber()->Value();
	}

	// Prototype functions
	static void Normalize(const FunctionCallbackInfo<Value>& info)
	{
		ScriptVector3* obj = ScriptObjectWrap::Unwrap<ScriptVector3>(info.This());
		obj->_vec.Normalize();
		return info.GetReturnValue().Set(info.This());
	}

private: // Data

	static v8::Persistent<v8::Function> _constructor;

	Vector3 _vec;
};
```

These wrappers can be a pain to do if you have a lot of classes to wraps so I suggest you find an automated process if possible. Maybe a mocing process would be nice to generate these wrappers using a MACRO system like QT. 

# API - Core systems

Core systems are a bit special. In Engine42, each core system is a singleton that can be accessed globally. Instead of making those globally accessible in JS just like the `console` object, I've decided to make them accessible through the `require` system. `require` is an idea took from node.js or RequireJS for which the function returns an exported module.

In example, to get the rendering core system, you would use `require` like this:

```js
function() {
	var renderer = require('renderer');
	renderer.drawText(...);
}
```

If you call `require('renderer')` multiple time, the same renderer core system instance will be returned. The first time `require('renderer')` is called, the object wrapper is created and cached for the entire session. That said, on the C++ side, core module are normal script object wrapper defined like the `ScriptVector3` class above. The only difference is that, these classes `::New*()` methods don't create a new instances of the core module but return the singleton instance.

# Require

It wouldn't be nice to code your entire game in a single JavaScript file, so this integration allows you to `require` other scripts from a given script.

For example, you can do something like this in Boot.js

```js
var _ = require('Scripts/Core/lodash'),
    StateMachine = require('Scripts/StateMachine'),
	entityCore = require('entity');
```

This example shows three different way of loading a module.

`_ = require('Scripts/Core/lodash'` loads the popular [lodash](https://lodash.com/) module and returns the lodash namespace in `_`. So in the current script, the user can use `_.*` to access any lodash functions.

`StateMachine = require('Scripts/StateMachine')` loads the `Scripts/StateMachine.js` file and returns the `StateMachine` constructor. Note that like `require` in node.js you don't need to specify the .js extension of the scripts.

`Scripts/StateMachine.js` is defined as:

```js
function StateMachine() {
	this._states = [];
}

StateMachine.prototype.states = function () {
	console.log('Hello from the state machine');

	return [1, 2, 3, 4];
}

module.exports = StateMachine;
``` 

So as you can see, everything you set in the `module.exports` object is exported to the caller. In practice here's what happens:

- `require` loads the JavaScript file content and stores it in a string buffer.
- Once the file is loaded, we wrap the file content in a special function that we will compile and evaluate.

```js
(function (__scriptFilePath) {
	var module = {
		exports: {}
		}, exports = module.exports;

	SCRIPT CONTENT INJECTED HERE;

	return module.exports;
}());
```

This way we make sure that nothing in the loaded script leaks in the global namespace and it allows us to easily catch what is exported in `module.exports`.

- Finally `entityCore = require('entity')` returns the wrapped core module instance. That means that the `require` implementation checks for specific strings when called and if a string matches a core module name, then that module instance is returned directly from cache, otherwise we proceed with the script loading. This is similar to many module in node.js. In example, in node.js you do `var fs = require('fs')` when you need access to the file system module. `'fs'` is not a normal user script, it is a system module loaded in a custom manner.

Note that once a script is loaded, the exported module is cached in memory so that next time the user `require` the same module or script file, the cached module is returned and not new instance. You can read more how modules are exported by convention [here](http://www.sitepoint.com/understanding-module-exports-exports-node-js/).

# Boot

V8 is initialized, we can load as many scripts as we want and all the API is exposed to V8. Now is the time to boot the machine. The boot method, loads the main script `Boot.js`, gets  persistent references to the `init`, `update`, `render` and `shutdown` function and calls `init()`. Once your game engine is initialize, you just need to call `ScriptEngine::Boot()` and you are set.

```cpp
void ScriptEngine::Boot()
{
	HandleScope handle_scope(mIsolate);
	v8::Context::Scope contextScope(Context());

	const VCNString bootScriptFilePath = "Boot.js";
	const VCNString& bootScriptContent = StringUtils::ReadFile(bootScriptFilePath);

	LoadScript(bootScriptFilePath, bootScriptContent);

	// Get access to required functions
	VCN_ASSERT_MSG(HasFunction("init"), "Boot script doesn't define init() callback.");
	VCN_ASSERT_MSG(HasFunction("update"), "Boot script doesn't define init() callback.");
	VCN_ASSERT_MSG(HasFunction("render"), "Boot script doesn't define init() callback.");
	VCN_ASSERT_MSG(HasFunction("shutdown"), "Boot script doesn't define init() callback.");

	mInitFunction = GetGlobalFunction("init");
	mUpdateFunction = GetGlobalFunction("update");
	mRenderFunction = GetGlobalFunction("render");
	mShutdownFunction = GetGlobalFunction("shutdown");

	// Call script initialization routine
	HandleScope scope(mIsolate);
	Local<Function> func = Local<Function>::New(mIsolate, mInitFunction);
	Handle<Value> args[1];
	func->Call(func->CreationContext()->Global(), 0, args);
}
```

Last four lines is the part that calls the JS `init()` method defined by the user in the main script.

# Update and Render

Update and render are called each frame to let the user update and render his game.

The `Update()` method does a bit more than just calling the `update` JS function. It manages the `setTimeout`, making sure that when a `setTimeout` times out it calls the JS callback.

```cpp
bool ScriptEngine::Update(const float elapsedTime)
{
	HandleScope scope(_isolate);

	#if ENABLE_V8_DEBUGGING
		v8::Context::Scope debugScope(Context());
	#endif

	Debug::ProcessDebugMessages();

	// Evaluate timeouts
	double currentTime = VCNSystem::GetInstance()->GetTotalElapsed();
	for(auto it = _timeouts.begin(), end = _timeouts.end(); it != end;)
	{
		Timeout& timeout = it->second;
		if (timeout.timeout <= currentTime)
		{
			Local<Function> func = Local<Function>::New(_isolate, timeout.func);
			Handle<Value> args[1];
			func->Call(func->CreationContext()->Global(), 0, args);

			_timeouts.erase(it++);
		}
		else
		{
			++it;
		}
	}

	Local<Function> func = Local<Function>::New(_isolate, _update_function);
	Handle<Value> args[1];
	args[0] = Number::New(_isolate, elapsedTime);
	Handle<Value> result = func->Call(func->CreationContext()->Global(), 1, args);

	return !result->IsBoolean() || result->BooleanValue();
}

void ScriptEngine::Render() const
{
	HandleScope scope(mIsolate);
	Context::Scope debugScope(Context());

	Local<Function> func = Local<Function>::New(mIsolate, mRenderFunction);
	Handle<Value> args[1];
	func->Call(func->CreationContext()->Global(), 0, args);
}
```

If you want to see how `setTimeout` callbacks are persisted, check the full source code below.

Maybe you've notice `v8::Debug::ProcessDebugMessages()`, what is that? We'll come back to this in the remote debugging section.

# Shutdown

Finally when the game is about to exit, we call the `shutdown()` function the same way we've called the `init()` function above.

# Remote Debugging

Any scripting integration, even in the best engine of the world, is useless if the scripts can't be debugged properly using a descent debugger. Since we don't want to write our own debug client, but we could, we allow remote debugging using an IDE like Webstorm.

## Socket server

To allow remote debugging we need to have a socket server that listens to the debugger client to handle debug commands. V8 has a really nice [debugging protocol that uses JSON](https://code.google.com/p/v8-wiki/wiki/DebuggerProtocol) and Webstorm can connect to your socket server to stream these JSON debug commands. So the only thing you need to do is to take debug client requests, forward them to V8, and listen to V8 debug message events and send them through sockets to the debug client. Also in the `ScriptEngine::Update()` method we call `v8::Debug::ProcessDebugMessages();` each frame so that V8 has a way to send debug messages/responses to our debug message handler in the main thread.

Here the socket server debug thread code:

```cpp
void ScriptEngine::DebuggerThread()
{
	WSADATA wsaData = {0};

	// Initialize Winsock
	int iResult = WSAStartup(MAKEWORD(2, 2), &wsaData);
	VCN_ASSERT_MSG(iResult == 0, "WSAStartup failed: %d", iResult);

	int sockfd, newsockfd, portno, clilen;
	struct sockaddr_in serv_addr, cli_addr;

	// Create the server socket
	sockfd = socket(AF_INET, SOCK_STREAM, 0);
	VCN_ASSERT_MSG(sockfd >= 0, "ERROR opening socket");

	// Listen to connections on port 42000
	::ZeroMemory((char *) &serv_addr, sizeof(serv_addr));
	portno = 42000;
	serv_addr.sin_family = AF_INET;
	serv_addr.sin_addr.s_addr = INADDR_ANY;
	serv_addr.sin_port = htons(portno);
	int bindResult = bind(sockfd, (struct sockaddr *) &serv_addr, sizeof(serv_addr));
	VCN_ASSERT_MSG(bindResult >= 0, "ERROR on binding");

	listen(sockfd,5);
	clilen = sizeof(cli_addr);

	while(1)
	{
		// Wait for debug client to connect.
		newsockfd = accept(sockfd, (struct sockaddr *) &cli_addr, &clilen);
		VCN_ASSERT_MSG(newsockfd >= 0, "ERROR on accept");

		// This is a HACK to store that last connected debug client.
		_main_debugger_socket = newsockfd;

		TRACE("Client connected to debugger.\n");

		// Say hello to the debug client. Webstorm needs that info in order to start a debugging session.
		SendBuffer(newsockfd, "Type: connect\r\n");
		SendBuffer(newsockfd, StringUtils::SPrintf("V8-Version: %s\r\n", v8::V8::GetVersion()));
		SendBuffer(newsockfd, "Protocol-Version: 1\r\n");
		SendBuffer(newsockfd, StringUtils::SPrintf("Embedding-Host: %s\r\n", "Engine42"));
		SendBuffer(newsockfd, "Content-Length: 0\r\n");
		SendBuffer(newsockfd, "\r\n");

		// We've said hello, now is time to get debug client requests, forward them to V8 and wait to debug message response.
		while (1)
		{
			// Get request
			VCNString request = GetRequest(newsockfd);

			if (request.empty())
				break;

			bool is_closing_session = (request.empty());

			if (is_closing_session)
			{
				// If we lost the connection, then simulate a disconnect msg:
				request = "{\"seq\":1,\"type\":\"request\",\"command\":\"disconnect\"}";
			}
			else
			{
				// Check if we're getting a disconnect request:
				const char* disconnectRequestStr =
					"\"type\":\"request\",\"command\":\"disconnect\"}";
				const char* result = strstr(request.c_str(), disconnectRequestStr);
				if (result != NULL)
				{
					is_closing_session = true;
				}
			}

			ScriptEngineClientData* gcd = new ScriptEngineClientData(this, newsockfd);

			// Send the request received to the debugger.
			auto data = ToUInt16Vector(request);
			v8::Debug::SendCommand(mIsolate, &data[0], data.size(), gcd);

			if (is_closing_session)
			{
				// Session is closed.
				break;
			}
		}

		TRACE("Client disconnected from debugger.\n");
	}

	WSACleanup();
}
```

The `ScriptEngineClientData` is used for the debug message handler to know which connection to forward the information too.

The debugging thread is composed of the following steps:

1- Initialize Winsock
2- Create your server socket
3- Listen to debug client connections
4- Get a debug client request
5- Dispatch the debug request to V8
6- If still connected goto step 4
7- If disconnected goto step 3

## Requests

Getting debug client request is trivial, you just need to read the client socket for a complete JSON message. First we read the message header that tells use what is the size of the message. The message length can be used to do some validation. Once we know the size of the message, then we read that amount of bytes to get all the stringify JSON object.

```cpp
VCNString GetRequest(int socket)
{
	int received;

	// Read header.
	int content_length = 0;
	while (true)
	{
		const int kHeaderBufferSize = 80;
		char header_buffer[kHeaderBufferSize];
		int header_buffer_position = 0;
		char c = '\0';  // One character receive buffer.
		char prev_c = '\0';  // Previous character.

		// Read until CRLF.
		while (!(c == '\n' && prev_c == '\r')) {
			prev_c = c;
			received = recv(socket, &c, 1, 0);
			VCN_ASSERT_MSG(received > 0, "ERROR reading from socket");

			// Add character to header buffer.
			if (header_buffer_position < kHeaderBufferSize) {
				header_buffer[header_buffer_position++] = c;
			}
		}

		// Check for end of header (empty header line).
		if (header_buffer_position == 2) {  // Receive buffer contains CRLF.
			break;
		}

		// Terminate header.
		VCN_ASSERT(header_buffer_position > 1);  // At least CRLF is received.
		VCN_ASSERT(header_buffer_position <= kHeaderBufferSize);
		header_buffer[header_buffer_position - 2] = '\0';

		// Split header.
		char* key = header_buffer;
		char* value = NULL;
		for (int i = 0; header_buffer[i] != '\0'; i++) {
			if (header_buffer[i] == ':') {
				header_buffer[i] = '\0';
				value = header_buffer + i + 1;
				while (*value == ' ') {
					value++;
				}
				break;
			}
		}

		// Check that key is Content-Length.
		if (strcmp(key, kContentLength) == 0) {
			// Get the content length value if present and within a sensible range.
			if (value == NULL || strlen(value) > 7) {
				return VCNString();
			}
			for (int i = 0; value[i] != '\0'; i++) {
				// Bail out if illegal data.
				if (value[i] < '0' || value[i] > '9') {
					return VCNString();
				}
				content_length = 10 * content_length + (value[i] - '0');
			}
		} else {
			// For now just print all other headers than Content-Length.
			TRACE("%s: %s\n", key, value != NULL ? value : "(no value)");
		}
	}

	// Return now if no body.
	if (content_length == 0) {
		return VCNString();
	}

	// Read body.
	VCNString buffer;
	buffer.resize(content_length);
	received = recv(socket, &buffer[0], content_length, 0);
	if (received < content_length) {
		TRACE("Error request data size\n");
		return VCNString();
	}
	buffer[content_length] = '\0';

	return buffer;
}
```

Messages are usually composed of

```
Content Length: XXXXX
\r\n
{.....}
```

The content length is in bytes.

## Responses

And when V8 calls your debug message handler, you just need to dispatch to the debug client socket.

The message has the follow form:

```
Content Length: XXXXX
\r\n
{.....}
```

The handler is:

```cpp
void ScriptEngine::DebuggerAgentMessageHandler(const Debug::Message& message)
{
	ScriptEngineClientData* cd = static_cast<ScriptEngineClientData*>(message.GetClientData());
	String::Utf8Value val(message.GetJSON());
	SendMessage(cd->socket, *val);
}
```

And send message is implemented as:

```cpp
void SendMessage(int conn, const VCNString& message)
{
	// Send the header.
	SendBuffer(conn, StringUtils::SPrintf("%s: %d\r\n", kContentLength, message.size()));

	// Terminate header with empty line.
	SendBuffer(conn, "\r\n");

	// Send message body as UTF-8.
	SendBuffer(conn, message);
}
```

## Webstorm use case

That's it! So using this you can fully debug your JS code using Webstorm!

The workflow is easy. Launch the game engine, once the debugger thread is active, you can connect to the V8 engine debugger using the port 42000. Here's are the step to do so:

- Launch the game engine.
- Start Webstorm, go to Run > Edit Configurations...
- Add a new one and use Node.js remote debugging (since node.js use the standard V8 debugging protcol that is exactly what we want)
![2015-03-05-12_03_48-Run_Debug-Configurations](/images/2015-03-05-12_03_48-Run_Debug-Configurations.png)
- Set any name you want set Host to be `localhost`.
- Set port to be 42000.
![2015-03-05-12_05_54-Run_Debug-Configurations](/images/2015-03-05-12_05_54-Run_Debug-Configurations.png)
- Press OK to save settings
- Press the debug icon.
![2015-03-05-12_06_48-Stingray-D__Engine42-WS-D__Engine42_Game_Data_Boot.js-WebStorm-9.0.3](/images/2015-03-05-12_06_48-Stingray-D__Engine42-WS-D__Engine42_Game_Data_Boot.js-WebStorm-9.0.3.png)
- Once connected, you'll the loaded scripts and the Connected to 127.0.0.1:42000 message.
![2015-03-05-12_08_00-Stingray-D__Engine42-WS-D__Engine42_Game_Data_Boot.js-WebStorm-9.0.3](/images/2015-03-05-12_08_00-Stingray-D__Engine42-WS-D__Engine42_Game_Data_Boot.js-WebStorm-9.0.3.png)
- Then just set a breakpoint and you'll see the Variables tab being updated! You can even use Webstorm's console to send live events to the game engine.
![2015-03-05-12_09_56-Stingray-D__Engine42-WS-D__Engine42_Game_Data_Boot.js-WebStorm-9.0.3.png](/images/2015-03-05-12_09_56-Stingray-D__Engine42-WS-D__Engine42_Game_Data_Boot.js-WebStorm-9.0.3.png.png)

I hope you've find this useful! Don't hesitate to take a look at the full sources and ask some questions.

In a future post I'll explain how hot-reloading works and how you can use it to good use.

Thanks

# Full source code

## ScriptEngine.h

```cpp
///
/// Copyright (C) 2015 - All Rights Reserved
/// All rights reserved. http://www.equals-forty-two.com
///
/// @brief V8 Script Engine interface
///

#pragma once

#include "Core/Types.h"
#include "Core/Singleton.h"
#include "Core/Thread.h"

#include <include/v8.h>

#include <map>

namespace v8{
	class Isolate;
}

typedef v8::Persistent<v8::Function, v8::CopyablePersistentTraits<v8::Function>> GlobalFunction;
typedef v8::Persistent<v8::Function, v8::NonCopyablePersistentTraits<v8::Function>> SavedFunction;

template <class TypeName, class CopyTrait>
inline v8::Local<TypeName> StrongPersistentToLocal(const v8::Persistent<TypeName, CopyTrait>& persistent)
{
	return *reinterpret_cast<v8::Local<TypeName>*>(const_cast<v8::Persistent<TypeName, CopyTrait>*>(&persistent));
}

class ScriptEngine : public Singleton<ScriptEngine>
{
public:

	ScriptEngine();
	~ScriptEngine();

	void Boot();
	bool Update(const float elapsedTime);
	void Render() const;
	void Shutdown();

	/// Returns the global context
	v8::Local<v8::Context> Context() { return StrongPersistentToLocal(_global_context); }
	v8::Local<v8::Context> Context() const { return StrongPersistentToLocal(_global_context); }

	/// Prints to console a JSON object.
	void PrintJson(v8::Handle<v8::Value> object);

private:

	struct Timeout
	{
		Timeout()
			: id(0)
			, timeout(std::numeric_limits<double>::max())
		{
		}
		Timeout(int id, double timeout, GlobalFunction func)
			: id(id)
			, timeout(timeout)
			, func(func)
		{
		}

		Timeout(const Timeout& t)
			: id(t.id)
			, timeout(t.timeout)
			, func(t.func)
		{
		}

		int id;
		double timeout;
		GlobalFunction func;
	};

	static ScriptEngine* GetDataScriptInterface(v8::Local<v8::Value> data);
	static ScriptEngine* GetDataScriptInterface(const v8::FunctionCallbackInfo<v8::Value>& args);
	static void DebuggerAgentMessageHandler(const v8::Debug::Message& message);

	/// Common functions
	static void Require(const v8::FunctionCallbackInfo<v8::Value>& args);
	static void SetTimeout(const v8::FunctionCallbackInfo<v8::Value>& args);
	static void ClearTimeout(const v8::FunctionCallbackInfo<v8::Value>& args);

	void InitializeV8();
	void UninitializeV8();

	bool HasFunction(const string& functionName);
	GlobalFunction GetGlobalFunction(const string& functionName);

	v8::Handle<v8::Value> LoadScript(const string& scriptFilePath, int script_line_offset = 0);
	v8::Handle<v8::Value> LoadScript(const string& scriptFilePath, const string& scriptSource, int script_line_offset = 0);

	bool ReportException(v8::Handle<v8::Value> er, v8::Handle<v8::Message> message);
	bool ReportException(const v8::TryCatch& try_catch);

	void DebuggerThread();
	void TestThread();

	v8::Platform* _v8_platform;
	v8::Isolate* _isolate;
	v8::Persistent<v8::Context> _global_context;
	v8::Local<v8::Object> _self;
	GlobalFunction _init_function;
	GlobalFunction _update_function;
	GlobalFunction _render_function;
	GlobalFunction _shutdown_function;
	int _next_timeout_id;
	std::map<int, Timeout> _timeouts;
	Thread _debugger_thread;
	std::map<string, v8::Persistent<v8::Value, v8::CopyablePersistentTraits<v8::Value>>> _module_cache;

	static int _main_debug_client_socket;
};
```

## ScriptEngine.cpp

```cpp
///
/// Copyright (C) 2015 - All Rights Reserved
/// All rights reserved. http://www.equals-forty-two.com
///
/// @brief V8 Script Engine implementation
///

#include "Precompiled.h"
#include "ScriptEngine.h"

#include "ScriptConsole.h"

#include "Core/Assert.h"
#include "Core/System.h"
#include "Core/StringUtils.h"
#include "Core/Vector.h"
#include "Core/Utilities.h"
#include "Core/Path.h"
#include "Logging/Log.h"

#include <Winsock2.h>

#if !defined(FINAL)
	#define ENABLE_V8_DEBUGGING 1
#endif

using namespace v8;

class ScriptObjectWrap
{
public:
	ScriptObjectWrap()
	{
		_refs = 0;
	}

	virtual ~ScriptObjectWrap()
	{
		if (persistent().IsEmpty())
			return;
		VCN_ASSERT(persistent().IsNearDeath());
		persistent().ClearWeak();
		persistent().Reset();
	}

	template <class T>
	static inline T* Unwrap(Handle<Object> handle)
	{
		VCN_ASSERT(!handle.IsEmpty());
		VCN_ASSERT(handle->InternalFieldCount() > 0);

		// Cast to ObjectWrap before casting to T.  A direct cast from void
		// to T won't work right when T has more than one base class.
		void* ptr = handle->GetAlignedPointerFromInternalField(0);
		ScriptObjectWrap* wrap = static_cast<ScriptObjectWrap*>(ptr);
		return static_cast<T*>(wrap);
	}

	inline Local<Object> handle()
	{
		return handle(Isolate::GetCurrent());
	}

	inline Local<Object> handle(Isolate* isolate)
	{
		return Local<Object>::New(isolate, persistent());
	}

	inline Persistent<Object>& persistent()
	{
		return _handle;
	}

protected:
	inline void Wrap(Handle<Object> handle)
	{
		VCN_ASSERT(persistent().IsEmpty());
		VCN_ASSERT(handle->InternalFieldCount() > 0);
		handle->SetAlignedPointerInInternalField(0, this);
		persistent().Reset(Isolate::GetCurrent(), handle);
		MakeWeak();
	}

	inline void MakeWeak(void)
	{
		persistent().SetWeak(this, WeakCallback);
		persistent().MarkIndependent();
	}

	/* Ref() marks the object as being attached to an event loop.
	* Refed objects will not be garbage collected, even if
	* all references are lost.
	*/
	virtual void Ref()
	{
		assert(!persistent().IsEmpty());
		persistent().ClearWeak();
		_refs++;
	}

	/* Unref() marks an object as detached from the event loop.  This is its
	* default state.  When an object with a "weak" reference changes from
	* attached to detached state it will be freed. Be careful not to access
	* the object after making this call as it might be gone!
	* (A "weak reference" means an object that only has a
	* persistent handle.)
	*
	* DO NOT CALL THIS FROM DESTRUCTOR
	*/
	virtual void Unref()
	{
		assert(!persistent().IsEmpty());
		assert(!persistent().IsWeak());
		assert(_refs > 0);
		if (--_refs == 0)
			MakeWeak();
	}

	int _refs;

private:
	static void WeakCallback(const WeakCallbackData<Object, ScriptObjectWrap>& data)
	{
		Isolate* isolate = data.GetIsolate();
		HandleScope scope(isolate);
		ScriptObjectWrap* wrap = data.GetParameter();
		VCN_ASSERT(wrap->_refs == 0);
		VCN_ASSERT(wrap->_handle.IsNearDeath());
		VCN_ASSERT(data.GetValue() == Local<Object>::New(isolate, wrap->_handle));
		wrap->_handle.Reset();
		delete wrap;
	}

	Persistent<Object> _handle;
};

class ScriptVector3 : public ScriptObjectWrap
{
public:

	/// Called to register the Vector3 constructor in the global namespace
	static void Create(Isolate* isolate, Local<Context> context)
	{
		// Prepare constructor template
		Local<FunctionTemplate> tpl = FunctionTemplate::New(isolate, New);
		tpl->SetClassName(String::NewFromUtf8(isolate, "Vector3"));
		tpl->InstanceTemplate()->SetInternalFieldCount(1);

		// Prototype
		tpl->PrototypeTemplate()->Set(String::NewFromUtf8(isolate, "normalize"), FunctionTemplate::New(isolate, Normalize)->GetFunction());

		// Properties
		tpl->InstanceTemplate()->SetAccessor(String::NewFromUtf8(isolate, "x"), GetX, SetX);
		tpl->InstanceTemplate()->SetAccessor(String::NewFromUtf8(isolate, "y"), GetY, SetY);
		tpl->InstanceTemplate()->SetAccessor(String::NewFromUtf8(isolate, "z"), GetZ, SetZ);

		// Define the constructor that will be used in JS to create a Vector3
		_constructor.Reset(isolate, tpl->GetFunction());

		context->Global()->Set(String::NewFromUtf8(isolate, "Vector3"), StrongPersistentToLocal(_constructor));
	}

	/// Called in C++ to create a new instance that can be passed to JS.
	static Handle<Value> NewInstance(Isolate* isolate, float x, float y, float z)
	{
		HandleScope scope(isolate);

		Handle<Function> constructor = StrongPersistentToLocal(_constructor);

		Local<Object> instance = constructor->NewInstance();
		ScriptVector3* obj = Unwrap<ScriptVector3>(instance);
		obj->_vec = Vector3(x, y, z);

		return instance;
	}

private:
	ScriptVector3(float x, float y, float z)
		: _vec(x, y, z)
	{
	}

	~ScriptVector3()
	{
	}

	// Constructor
	static void New(const FunctionCallbackInfo<Value>& info)
	{
		HandleScope scope(info.GetIsolate());

		VCN_ASSERT(info.Length() == 3);
		VCN_ASSERT(info[0]->IsNumber());
		VCN_ASSERT(info[1]->IsNumber());
		VCN_ASSERT(info[2]->IsNumber());

		float x = static_cast<float>(info[0]->ToNumber()->Value());
		float y = static_cast<float>(info[0]->ToNumber()->Value());
		float z = static_cast<float>(info[0]->ToNumber()->Value());

		ScriptVector3* obj = new ScriptVector3(x, y, z);
		obj->Wrap(info.This());

		return info.GetReturnValue().Set(info.This());
	}

	// Accessors
	static void GetX(Local<String> /*property*/, const PropertyCallbackInfo<Value>& info)
	{
		const ScriptVector3* obj = Unwrap<ScriptVector3>(info.This());
		info.GetReturnValue().Set(Number::New(info.GetIsolate(), obj->_vec.x));
	}
	static void GetY(Local<String> /*property*/, const PropertyCallbackInfo<Value>& info)
	{
		const ScriptVector3* obj = Unwrap<ScriptVector3>(info.This());
		info.GetReturnValue().Set(Number::New(info.GetIsolate(), obj->_vec.y));
	}
	static void GetZ(Local<String> /*property*/, const PropertyCallbackInfo<Value>& info)
	{
		const ScriptVector3* obj = Unwrap<ScriptVector3>(info.This());
		info.GetReturnValue().Set(Number::New(info.GetIsolate(), obj->_vec.z));
	}

	static void SetX(Local<String> /*property*/, Local<Value> value, const PropertyCallbackInfo<void>& info)
	{
		ScriptVector3* obj = Unwrap<ScriptVector3>(info.This());
		obj->_vec.x = static_cast<float>(value->ToNumber()->Value());
	}
	static void SetY(Local<String> /*property*/, Local<Value> value, const PropertyCallbackInfo<void>& info)
	{
		ScriptVector3* obj = Unwrap<ScriptVector3>(info.This());
		obj->_vec.y = static_cast<float>(value->ToNumber()->Value());
	}
	static void SetZ(Local<String> /*property*/, Local<Value> value, const PropertyCallbackInfo<void>& info)
	{
		ScriptVector3* obj = Unwrap<ScriptVector3>(info.This());
		obj->_vec.z = static_cast<float>(value->ToNumber()->Value());
	}

	// Prototype functions
	static void Normalize(const FunctionCallbackInfo<Value>& info)
	{
		ScriptVector3* obj = Unwrap<ScriptVector3>(info.This());
		obj->_vec.Normalize();
		return info.GetReturnValue().Set(info.This());
	}

private: // Data

	static Persistent<Function> _constructor;

	Vector3 _vec;
};

Persistent<Function> ScriptVector3::_constructor;

namespace {

	const int debugging_port = 5858;
	const char* const kContentLength = "Content-Length";
	const int kContentLengthSize = strlen(kContentLength);

	class MallocArrayBufferAllocator : public ArrayBuffer::Allocator {
	public:
		virtual void* Allocate(size_t length) override { return malloc(length); }
		virtual void* AllocateUninitialized(size_t length) override { return malloc(length); }
		virtual void Free(void* data, size_t /*length*/) override { free(data); }
	};

	class ScriptEngineClientData : public Debug::ClientData
	{
	public:
		ScriptEngineClientData(ScriptEngine* _this, int socket)
			: engine(_this)
			, socket(socket)
		{
		}

		ScriptEngine* engine;
		int socket;
	};

	void SendBuffer(int socket, const string& message)
	{
		int n = send(socket, message.c_str(), message.size(), 0);
		VCN_ASSERT_MSG(n >= 0, "ERROR writing to socket (%d)", WSAGetLastError());
	}

	void SendMessage(int conn, const string& message)
	{
		// Send the header.
		SendBuffer(conn, StringUtils::SPrintf("%s: %d\r\n", kContentLength, message.size()));

		// Terminate header with empty line.
		SendBuffer(conn, "\r\n");

		// Send message body as UTF-8.
		SendBuffer(conn, message);
	}

	string GetRequest(int socket)
	{
		int received;

		// Read header.
		int content_length = 0;
		while (true)
		{
			const int kHeaderBufferSize = 80;
			char header_buffer[kHeaderBufferSize];
			int header_buffer_position = 0;
			char c = '\0';  // One character receive buffer.
			char prev_c = '\0';  // Previous character.

			// Read until CRLF.
			while (!(c == '\n' && prev_c == '\r'))
			{
				prev_c = c;
				received = recv(socket, &c, 1, 0);
				int wsa_error = WSAGetLastError();
				if (wsa_error == WSAECONNRESET)
					return string();
				VCN_ASSERT_MSG(received >= 0, "ERROR reading from socket (%d)", wsa_error);

				// Add character to header buffer.
				if (header_buffer_position < kHeaderBufferSize) {
					header_buffer[header_buffer_position++] = c;
				}
			}

			// Check for end of header (empty header line).
			// Receive buffer contains CRLF.
			if (header_buffer_position == 2)
			{
				break;
			}

			// Terminate header.
			VCN_ASSERT(header_buffer_position > 1);  // At least CRLF is received.
			VCN_ASSERT(header_buffer_position <= kHeaderBufferSize);
			header_buffer[header_buffer_position - 2] = '\0';

			// Split header.
			char* key = header_buffer;
			char* value = nullptr;
			for (int i = 0; header_buffer[i] != '\0'; i++)
			{
				if (header_buffer[i] == ':')
				{
					header_buffer[i] = '\0';
					value = header_buffer + i + 1;
					while (*value == ' ')
					{
						value++;
					}
					break;
				}
			}

			// Check that key is Content-Length.
			if (strcmp(key, kContentLength) == 0)
			{
				// Get the content length value if present and within a sensible range.
				if (value == nullptr || strlen(value) > 7)
				{
					return string();
				}
				for (int i = 0; value[i] != '\0'; i++)
				{
					// Bail out if illegal data.
					if (value[i] < '0' || value[i] > '9')
					{
						return string();
					}
					content_length = 10 * content_length + (value[i] - '0');
				}
			}
			else
			{
				// For now just print all other headers than Content-Length.
				TRACE("%s: %s\n", key, value != nullptr ? value : "(no value)");
			}
		}

		// Return now if no body.
		if (content_length == 0)
		{
			return string();
		}

		// Read body.
		string buffer;
		buffer.resize(content_length);
		received = recv(socket, &buffer[0], content_length, 0);
		if (received < content_length)
		{
			TRACE("Error request data size\n");
			return string();
		}
		buffer[content_length] = '\0';

		return buffer;
	}

	template<typename S>
	std::vector<uint16_t> ToUInt16Vector(const S& str)
	{
		size_t s = str.size();
		std::vector<uint16_t> data(s);

		for(size_t i = 0; i < s; ++i)
		{
			if (str[i] != 0)
				data[i] = static_cast<uint16_t>(str[i]);
			else
				data[i] = ' ';
		}

		return data;
	}
}

int ScriptEngine::_main_debug_client_socket = -1;

ScriptEngine::ScriptEngine()
	: _v8_platform(nullptr)
	, _next_timeout_id(0)
	, _debugger_thread("V8 Debugger Thread")
{
	InitializeV8();
}

ScriptEngine::~ScriptEngine()
{
	UninitializeV8();
}

ScriptEngine* ScriptEngine::GetDataScriptInterface(const FunctionCallbackInfo<Value>& args)
{
	return GetDataScriptInterface(args.Data());
}

ScriptEngine* ScriptEngine::GetDataScriptInterface(Local<Value> data)
{
	Local<Object> self = data.As<Object>();
	Local<External> wrap = Local<External>::Cast(self->GetInternalField(0));
	return static_cast<ScriptEngine*>(wrap->Value());
}

void ScriptEngine::Require(const FunctionCallbackInfo<Value>& args)
{
	VCN_ASSERT(args.Length() == 1 && args[0]->IsString());

	HandleScope scope(args.GetIsolate());
	ScriptEngine* script_engine = GetDataScriptInterface(args);

	String::Utf8Value required_module(args[0]);

	// TODO: Check if system module (i.e. fs, net, ...)
	// TODO: Check if core module (i.e. entity, renderer, resource, etc.)

	// Get current script filepath
	String::Utf8Value current_script_filepath(StackTrace::CurrentStackTrace(args.GetIsolate(), 1, StackTrace::kScriptName)->GetFrame(0)->GetScriptName());
	Path current_script_path(*current_script_filepath);
	Path script_path = current_script_path.GetParent().Join(*required_module);

	string script_filepath = script_path;
	if (script_filepath.find(".js") == string::npos)
		script_filepath += ".js";

	// Is module already in cache?
	auto cached_module_itr = script_engine->_module_cache.find(script_filepath);
	if (cached_module_itr != script_engine->_module_cache.end())
	{
		TRACE("Loaded module %s from cache\n", script_filepath.c_str());
		args.GetReturnValue().Set(StrongPersistentToLocal(cached_module_itr->second));
		return;
	}

	string fileContent = StringUtils::ReadFile(script_filepath);
	if(fileContent.length() > 0)
	{
		StringBuilder sb;
		sb << "\
			  (function (__scriptFilePath) {\n\
			  var module = {\n\
			  exports: {}\n\
			  }, exports = module.exports;\n" << fileContent << "\n;return module.exports;}('" << script_filepath << "'));";

		Local<Value> exports = script_engine->LoadScript(script_filepath, sb.str().c_str(), 4);

		// Store module in cache
		script_engine->_module_cache[script_filepath].Reset(args.GetIsolate(), exports);

		args.GetReturnValue().Set(exports);
	}
	else
	{
		args.GetReturnValue().Set(Undefined(args.GetIsolate()));
	}
}

void ScriptEngine::SetTimeout(const FunctionCallbackInfo<Value>& args)
{
	VCN_ASSERT(args.Length() >= 2);
	VCN_ASSERT(!args[0].As<Function>().IsEmpty());
	VCN_ASSERT(!args[1].As<Number>().IsEmpty());

	HandleScope scope(args.GetIsolate());
	ScriptEngine* script_interface = GetDataScriptInterface(args);

	int timeoutId = script_interface->_next_timeout_id++;

	Local<Function> callback = args[0].As<Function>();
	double timeout_ms = args[1].As<Number>()->NumberValue();

	script_interface->_timeouts[timeoutId] = Timeout(
		timeoutId,
		VCNSystem::GetInstance()->GetTotalElapsed() + (timeout_ms / 1000.0),
		GlobalFunction(args.GetIsolate(), callback));

	args.GetReturnValue().Set(Number::New(args.GetIsolate(), timeoutId));
}

void ScriptEngine::ClearTimeout(const FunctionCallbackInfo<Value>& args)
{
	VCN_ASSERT(args.Length() == 1);
	VCN_ASSERT(args[0]->IsNumber());

	ScriptEngine* script_engine = GetDataScriptInterface(args);

	int timeout_id = args[0]->ToNumber()->ToInt32()->Value();
	auto timeout_itr = script_engine->_timeouts.find(timeout_id);
	if (timeout_itr != script_engine->_timeouts.end())
	{
		script_engine->_timeouts.erase(timeout_itr);
	}
}

Handle<Value> ScriptEngine::LoadScript(const string& script_file_path, const string& script_source, int script_line_offset /*= 0*/)
{
	char fullPath[MAX_PATH];
	_fullpath(fullPath, script_file_path.c_str(), MAX_PATH);

	TryCatch try_catch;
	try_catch.SetVerbose(true);
	Handle<String> source = String::NewFromUtf8(_isolate, script_source.c_str());
	ScriptOrigin scriptOrigin(String::NewFromUtf8(_isolate, fullPath), Integer::New(_isolate, -script_line_offset));
	Handle<Script> script = Script::Compile(source, &scriptOrigin);
	if (script.IsEmpty())
	{
		if (!ReportException(try_catch))
			exit(3);
	}

	Local<Value> v = script->Run();
	if (v.IsEmpty())
	{
		if (!ReportException(try_catch))
			exit(4);
	}

	return v;
}

Handle<Value> ScriptEngine::LoadScript(const string& script_file_path, int script_line_offset /*= 0*/)
{
	if (!VCN::FileExists(script_file_path))
		return Undefined(_isolate);
	return LoadScript(script_file_path, StringUtils::ReadFile(script_file_path), script_line_offset);
}

void ScriptEngine::InitializeV8()
{
	// Initialize V8.
	V8::InitializeICU();
	_v8_platform = platform::CreateDefaultPlatform();
	V8::InitializePlatform(_v8_platform);
	V8::Initialize();

	string v8_flags = "--harmony-scoping";
	#if ENABLE_V8_DEBUGGING
		v8_flags += " --debugger --expose_debug_as=v8debug";
	#endif
	V8::SetFlagsFromString(v8_flags.c_str(), v8_flags.size());

	// Define a custom allocator for V8 to allocate some memory when needed.
	V8::SetArrayBufferAllocator(new MallocArrayBufferAllocator);

	// Create a new Isolate and make it the current one.
	_isolate = Isolate::New();
	_isolate->Enter();

	// Create a stack-allocated handle scope.
	HandleScope scope(_isolate);

	// Create a new context.
	Local<v8::Context> context = Context::New(_isolate);
	_global_context.Reset(_isolate, context);
	v8::Context::Scope contextScope(context);

	// Create local to self
	Handle<ObjectTemplate> t = ObjectTemplate::New();
	t->SetInternalFieldCount(1);
	_self = t->NewInstance();
	_self->SetInternalField(0, External::New(_isolate, this));

#if ENABLE_V8_DEBUGGING
	// Enable the debugger thread to support remote debugging.
	Debug::SetMessageHandler(DebuggerAgentMessageHandler);
	_debugger_thread.EnqueueFunctionCall(FunctionCall(&ScriptEngine::DebuggerThread, *this));
#endif

	// Expose main systems to the scripts
	Console::Create(_isolate, context);
	ScriptVector3::Create(_isolate, context);
#if 0
	ScriptWorldCore::Create(mIsolate, context);
	ScriptEntityCore::Create(mIsolate, context);
	ScriptRenderCore::Create(mIsolate, context);
	ScriptQuaternion::Create(mIsolate, context);
	ScriptEntity::Create(mIsolate, context);
	ScriptComponent::Create(mIsolate, context);
#endif

	// Defines common JavaScript global functions
	context->Global()->Set(String::NewFromUtf8(_isolate, "require"), FunctionTemplate::New(_isolate, Require, _self)->GetFunction());
	context->Global()->Set(String::NewFromUtf8(_isolate, "setTimeout"), FunctionTemplate::New(_isolate, SetTimeout, _self)->GetFunction());
	context->Global()->Set(String::NewFromUtf8(_isolate, "clearTimeout"), FunctionTemplate::New(_isolate, ClearTimeout, _self)->GetFunction());
}

void ScriptEngine::UninitializeV8()
{
	_module_cache.clear();

	_init_function = GlobalFunction();
	_update_function = GlobalFunction();
	_render_function = GlobalFunction();
	_shutdown_function = GlobalFunction();

	// Dispose the isolate and tear down V8.
	_isolate->Exit();
	_isolate->Dispose();
	_isolate = nullptr;

	V8::Dispose();
	V8::ShutdownPlatform();
	delete _v8_platform;
}

void ScriptEngine::Boot()
{
	HandleScope handle_scope(_isolate);

	#if ENABLE_V8_DEBUGGING
		v8::Context::Scope contextScope(Context());
	#endif

	// Load core libraries
	LoadScript("Scripts/Core/window.js");
	LoadScript("Scripts/Core/document.js");

	const string bootScriptFilePath = "Scripts/main.js";
	const string& bootScriptContent = StringUtils::ReadFile(bootScriptFilePath);

	LoadScript(bootScriptFilePath, bootScriptContent);

	// Get access to main functions
	_init_function = GetGlobalFunction("init");
	_update_function = GetGlobalFunction("update");
	_render_function = GetGlobalFunction("render");
	_shutdown_function = GetGlobalFunction("shutdown");

	// Call script initialization routine
	HandleScope scope(_isolate);
	Local<Function> func = Local<Function>::New(_isolate, _init_function);
	Handle<Value> args[1];
	func->Call(func->CreationContext()->Global(), 0, args);
}

bool ScriptEngine::HasFunction(const string& functionName)
{
	Handle<Object> global = Context()->Global();
	Handle<Value> value = global->Get(String::NewFromUtf8(_isolate, functionName.c_str()));
	return value->IsFunction();
}

GlobalFunction ScriptEngine::GetGlobalFunction(const string& functionName)
{
	Handle<Object> global = _isolate->GetCurrentContext()->Global();
	Handle<Value> value = global->Get(String::NewFromUtf8(_isolate, functionName.c_str()));

	VCN_ASSERT_MSG(value->IsFunction(), "Function %s doesn't exist.", functionName.c_str());
	return GlobalFunction(_isolate, Handle<Function>::Cast(value));
}

bool ScriptEngine::Update(const float elapsedTime)
{
	HandleScope scope(_isolate);

	#if ENABLE_V8_DEBUGGING
		v8::Context::Scope debugScope(Context());
	#endif

	Debug::ProcessDebugMessages();

	// Evaluate timeouts
	double currentTime = VCNSystem::GetInstance()->GetTotalElapsed();
	for(auto it = _timeouts.begin(), end = _timeouts.end(); it != end;)
	{
		Timeout& timeout = it->second;
		if (timeout.timeout <= currentTime)
		{
			Local<Function> func = Local<Function>::New(_isolate, timeout.func);
			Handle<Value> args[1];
			func->Call(func->CreationContext()->Global(), 0, args);

			_timeouts.erase(it++);
		}
		else
		{
			++it;
		}
	}

	Local<Function> func = Local<Function>::New(_isolate, _update_function);
	Handle<Value> args[1];
	args[0] = Number::New(_isolate, elapsedTime);
	Handle<Value> result = func->Call(func->CreationContext()->Global(), 1, args);

	return !result->IsBoolean() || result->BooleanValue();
}

void ScriptEngine::Render() const
{
	HandleScope scope(_isolate);

	#if ENABLE_V8_DEBUGGING
		v8::Context::Scope debugScope(Context());
	#endif

	Local<Function> func = Local<Function>::New(_isolate, _render_function);
	Handle<Value> args[1];
	func->Call(func->CreationContext()->Global(), 0, args);
}

void ScriptEngine::Shutdown()
{
	HandleScope scope(_isolate);
	Context::Scope debugScope(Context());

	Local<Function> func = Local<Function>::New(_isolate, _shutdown_function);
	Handle<Value> args[1];
	func->Call(func->CreationContext()->Global(), 0, args);
}

bool ScriptEngine::ReportException(Handle<Value> er, Handle<Message> message)
{
	HandleScope scope(_isolate);

	Local<Value> trace_value;

	if (er->IsUndefined() || er->IsNull())
		trace_value = Undefined(_isolate);
	else
		trace_value = er->ToObject()->Get(String::NewFromUtf8(_isolate, "stack"));

	String::Utf8Value trace(trace_value);

	// range errors have a trace member set to undefined
	if (trace.length() > 0 && !trace_value->IsUndefined())
	{
		String::Utf8Value source_line(message->GetSourceLine());
		bool ignore = false;
		return DisplayAssert(0, *source_line, message->GetLineNumber(),
			*String::Utf8Value(message->GetScriptResourceName()), ignore, *trace);
	}

	// this really only happens for RangeErrors, since they're the only
	// kind that won't have all this info in the trace, or when non-Error
	// objects are thrown manually.
	Local<Value> message_inner;
	Local<Value> name;

	if (er->IsObject())
	{
		Local<Object> err_obj = er.As<Object>();
		message_inner = err_obj->Get(String::NewFromUtf8(_isolate, "message"));
		name = err_obj->Get(String::NewFromUtf8(_isolate, "name"));
	}

	if (message_inner.IsEmpty() ||
		message_inner->IsUndefined() ||
		name.IsEmpty() ||
		name->IsUndefined())
	{
		// Not an error object. Just print as-is.
		String::Utf8Value message_string(er);

		bool dummy = false;
		return DisplayAssert(0, *message_string, message->GetLineNumber(),
			*String::Utf8Value(message->GetScriptResourceName()), dummy, "Script Exception");
	}

	String::Utf8Value name_string(name);
	String::Utf8Value message_string(message_inner);

	bool dummy = false;
	return DisplayAssert(0, *message_string, message->GetLineNumber(),
		*String::Utf8Value(message->GetScriptResourceName()), dummy, *name_string);
}

bool ScriptEngine::ReportException(const TryCatch& try_catch)
{
	return ReportException(try_catch.Exception(), try_catch.Message());
}

void ScriptEngine::PrintJson(Handle<Value> object)
{
	HandleScope scope(_isolate);

	if (object->IsFunction())
		return;

	Handle<Object> global = Context()->Global();

	Handle<Object> JSON = global->Get(String::NewFromUtf8(_isolate, "JSON"))->ToObject();
	Handle<Function> JSON_stringify = Handle<Function>::Cast(JSON->Get(String::NewFromUtf8(_isolate, "stringify")));

	Handle<Value> args[1];
	args[0] = object;
	VCNLog << *String::Utf8Value(JSON_stringify->Call(JSON, 1, args)) << std::endl;
}

// Public V8 debugger API message handler function. This function just delegates
// to the debugger agent through it's data parameter.
void ScriptEngine::DebuggerAgentMessageHandler(const Debug::Message& message)
{
	int socket = _main_debug_client_socket;
	ScriptEngineClientData* cd = static_cast<ScriptEngineClientData*>(message.GetClientData());
	if (cd != nullptr)
		socket = cd->socket;

	if (socket <= 0)
		return;

	String::Utf8Value val(message.GetJSON());
	SendMessage(socket, *val);
}

void ScriptEngine::DebuggerThread()
{
	WSADATA wsa_data = {0};

	// Initialize Winsock
	int ws_result = WSAStartup(MAKEWORD(2, 2), &wsa_data);
	VCN_ASSERT_MSG(ws_result == 0, "WSAStartup failed: %d", ws_result);

	int sockfd, client_socket, portno, clilen;
	sockaddr_in serv_addr, cli_addr;

	sockfd = socket(AF_INET, SOCK_STREAM, 0);
	VCN_ASSERT_MSG(sockfd >= 0, "ERROR opening socket (%d)", WSAGetLastError());

	::ZeroMemory(reinterpret_cast<char *>(&serv_addr), sizeof(serv_addr));
	portno = debugging_port;
	serv_addr.sin_family = AF_INET;
	serv_addr.sin_addr.s_addr = INADDR_ANY;
	serv_addr.sin_port = htons(portno);
	int bindResult = bind(sockfd, reinterpret_cast<sockaddr *>(&serv_addr), sizeof(serv_addr));
	VCN_ASSERT_MSG(bindResult >= 0, "ERROR on binding (%d)", WSAGetLastError());

	listen(sockfd,5);
	clilen = sizeof(cli_addr);

	while(1)
	{
		client_socket = accept(sockfd, reinterpret_cast<sockaddr *>(&cli_addr), &clilen);
		VCN_ASSERT_MSG(client_socket >= 0, "ERROR on accept (%d)", WSAGetLastError());

		_main_debug_client_socket = client_socket;

		TRACE("Client connected to debugger.\n");

		// Say hello
		SendBuffer(client_socket, "Type: connect\r\n");
		SendBuffer(client_socket, StringUtils::SPrintf("V8-Version: %s\r\n", V8::GetVersion()));
		SendBuffer(client_socket, "Protocol-Version: 1\r\n");
		SendBuffer(client_socket, StringUtils::SPrintf("Embedding-Host: %s\r\n", "Engine42"));
		SendBuffer(client_socket, "Content-Length: 0\r\n");
		SendBuffer(client_socket, "\r\n");

		while (1)
		{
			// Get request
			string request = GetRequest(client_socket);

			if (request.empty())
				break;

			bool is_closing_session = (request.empty());

			if (is_closing_session)
			{
				// If we lost the connection, then simulate a disconnect msg:
				request = "{\"seq\":1,\"type\":\"request\",\"command\":\"disconnect\"}";
			}
			else
			{
				// Check if we're getting a disconnect request:
				const char* disconnect_request_str = "\"type\":\"request\",\"command\":\"disconnect\"}";
				const char* result = strstr(request.c_str(), disconnect_request_str);
				if (result != nullptr)
				{
					is_closing_session = true;
				}
			}

			// Send the request received to the debugger.
			auto data = ToUInt16Vector(request);
			Debug::SendCommand(_isolate, &data[0], data.size(), new ScriptEngineClientData(this, client_socket));

			if (is_closing_session)
			{
				// Session is closed.
				break;
			}
		}

		TRACE("Client disconnected from debugger.\n");
	}

	WSACleanup();
}
```

## ScriptConsole.h

```cpp
///
/// Copyright (C) 2015 - All Rights Reserved
/// All rights reserved. http://www.equals-forty-two.com
///
/// @brief V8 console.* interface
///

#pragma once

#include "Core/Types.h"

#include <include/v8.h>

namespace v8{
	class Isolate;
}

class Console
{
public:
	static  void Create(v8::Isolate* isolate, v8::Local<v8::Context> context);

	static void Print(const v8::FunctionCallbackInfo<v8::Value>& args);
	static void Assert(const v8::FunctionCallbackInfo<v8::Value>& args);
	static void VoidMessage(const v8::FunctionCallbackInfo<v8::Value>& args);

private:

	static string StringifyArguments(const v8::FunctionCallbackInfo<v8::Value>& args, int start = 0, int end = -1);
};
```

## ScriptConsole.cpp

```cpp
///
/// Copyright (C) 2015 - All Rights Reserved
/// All rights reserved. http://www.equals-forty-two.com
///
/// @brief V8 console.* implementation
///

#include "Precompiled.h"
#include "ScriptConsole.h"

#include "Core/Assert.h"
#include "Logging/Log.h"

using namespace v8;

void Console::Create(Isolate* isolate, Local<Context> context)
{
	Handle<ObjectTemplate> console = ObjectTemplate::New(isolate);
	console->Set(String::NewFromUtf8(isolate, "log"), FunctionTemplate::New(isolate, Print));
	console->Set(String::NewFromUtf8(isolate, "warn"), FunctionTemplate::New(isolate, Print));
	console->Set(String::NewFromUtf8(isolate, "info"), FunctionTemplate::New(isolate, Print));
	console->Set(String::NewFromUtf8(isolate, "error"), FunctionTemplate::New(isolate, Print));
	console->Set(String::NewFromUtf8(isolate, "assert"), FunctionTemplate::New(isolate, Assert));
	console->Set(String::NewFromUtf8(isolate, "void"), FunctionTemplate::New(isolate, VoidMessage));
	context->Global()->Set(String::NewFromUtf8(isolate, "console"), console->NewInstance());
}

void Console::Print(const FunctionCallbackInfo<Value>& args)
{
	VCNLog << "[Script Console] " << StringifyArguments(args) << std::endl;
	args.GetReturnValue().Set(args.Holder());
}

void Console::Assert(const FunctionCallbackInfo<Value>& args)
{
	VCN_ASSERT(args.Length() >= 1);

	bool to_assert = args[0]->IsFalse();
	if (to_assert)
	{
		const string& assertion_msg = StringifyArguments(args, 1);

		// Get script information
		auto stack_frame = StackTrace::CurrentStackTrace(args.GetIsolate(), 1, StackTrace::kOverview)->GetFrame(0);
		auto filename = stack_frame->GetScriptName();
		int line_number = stack_frame->GetLineNumber();
		String::Utf8Value script_name(filename);

		bool ignore = false;
		DisplayAssert(false, assertion_msg.c_str(), line_number, *script_name, ignore, "");
	}

	args.GetReturnValue().Set(args.Holder());
}

void Console::VoidMessage(const FunctionCallbackInfo<Value>& args)
{
	args.GetReturnValue().Set(args.Holder());
}

string Console::StringifyArguments(const FunctionCallbackInfo<Value>& args, int start /*= 0*/, int end /*= -1*/ )
{
	StringBuilder sb;
	end = (end == -1 ? args.Length() : end);
	for (int i = start; i != end; ++i)
	{
		if (i > start)
			sb << " ";
		String::Utf8Value str(args[i]);
		sb << *str;
	}

	return sb.str();
}
```

## main.js (example)

```js
///
/// Copyright (C) 2015 - All Rights Reserved
/// All rights reserved. http://www.equals-forty-two.com
///
/// @brief Main game entry point.
///

var _ = require('./3rdparty/lodash'),/*,
    Prototype = require('./3rdparty/prototype'),*/
    StateMachine = require('./statemachine');

var _reloadCount = _reloadCount ? _reloadCount + 1 : 0;
var _gameStateMachine = _gameStateMachine || null;

console.log('Script reloaded:', _reloadCount);

/**
 * Called when the game is initialized.
 * @static
 * @global
 */
function init() {
    'use strict';

    var innerVar = 24;
    var unwantedTimeoutId = setTimeout(function () {
        console.assert(false, 'This should not happen!');
    }, 10000);

    setTimeout(function () {
        var localVar = 99;
        console.log('local var', localVar, 'Inner var', innerVar, 'Global var', _reloadCount);

        clearTimeout(unwantedTimeoutId);
    }, 2000);

    // Create the game state machine
    _gameStateMachine = new StateMachine();
    _.each(_gameStateMachine.states(), function (v) {
        console.log(v);
    });

    console.log('Init');
}

/**
 * Called each frame.
 * @param dt {Number} elapsed time since last time the function was called.
 */
function update(dt) {
    'use strict';

    var sum = 0;
    for (let i = 0; i < dt * 100000; ++i) {
        sum += i;
        sum *= i / 2;
        sum /= 1.4;
    }

    if (!_gameStateMachine.printed) {
        _gameStateMachine.printStates();
        _gameStateMachine.printed = true;
    }
}

/**
 * Called each frame to render the game.
 */
function render() {
    'use strict';
}

/**
 * Called when the application about to shutdown.
 */
function shutdown() {
    'use strict';
    console.log('Shutdown');
}
```
