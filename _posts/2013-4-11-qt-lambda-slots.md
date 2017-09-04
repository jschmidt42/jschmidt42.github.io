---
layout: post
title: QT lambda slots
category: code
tags: [qt, lamdba, c++]
---

I've toyed with QT connections trying to create anonymous function when connecting a slot to a signal. For instance, I am trying to do the following:

```cpp
menu->addAction( QIcon("some_icon.png"), "Open", this, SLOT(OnOpenTriggered()) );
```

So far so good, but sometime I find it cumbersome to define all the slots. Create a declaration, a definition, etc...

I would like to declare the body right there! Here's what I would like to do using *std::function* or a [c++ lambda](http://stackoverflow.com/questions/7627098/what-is-a-lambda-expression-in-c11):

```cpp
menu->addAction( QIcon("some_icon.png"), "Open", _Q, _Q->Call( [this](){
    // body here...
    this->showNormal(); // e.g.
} ) );
```

I've achieved that by creating a global object named _Q which is responsible to maintained the QT meta object information about all connection that needs to be called.

Here's the header of the class in question:

```cpp
class GlobalQTReceiver : public QObject
{
	Q_GADGET;
public:

	typedef std::function<void()> Func;

	GlobalQTReceiver();

	const char* Call(Func slotCallback);

protected:

	virtual const QMetaObject *metaObject() const override;
	virtual int qt_metacall(QMetaObject::Call, int, void **) override;
	virtual void* qt_metacast(const char *_clname) override;
	
protected Q_SLOTS:

	void func1();

private:

	struct FuncInfo {
		std::string name;
		std::string slotName;
		Func callback;
	};

	void FillMetaStructs();

	int mNextId;
	std::vector<FuncInfo> mFuncs;
	std::vector<uint> mMetaData;
	std::vector<char> mMetaString;
	QMetaObject mMetaObject;
};

extern QTUtils::Internal::GlobalQTReceiver* _Q;
```

And here's the implementation:

```cpp

GlobalQTReceiver::GlobalQTReceiver()
	: mNextId(0)
{
	FillMetaStructs();
}

const char* GlobalQTReceiver::Call( std::function<void()> slotCallback )
{
	std::stringstream ss;
	ss << "func" << mNextId++ << "()";

	FuncInfo fi;
	fi.name = ss.str();
	fi.slotName = "1"+fi.name;
	fi.callback = slotCallback;

	mFuncs.push_back( fi );
	FillMetaStructs();
				
	return mFuncs.back().slotName.c_str();
}

int GlobalQTReceiver::qt_metacall(QMetaObject::Call call, int id, void** args)
{
	Q_ASSERT( call == QMetaObject::InvokeMetaMethod );

	id = QObject::qt_metacall(call, id, args);
	if (id < 0)
		return id;
	
	Func slotCallback = mFuncs[id].callback;
	id -= mFuncs.size();

	slotCallback();

	return id;
}

void* GlobalQTReceiver::qt_metacast( const char *_clname )
{
	Q_ASSERT( !"not supported" );
	return nullptr;
}

const QMetaObject* GlobalQTReceiver::metaObject() const 
{
	return QObject::d_ptr->metaObject ? QObject::d_ptr->metaObject : &mMetaObject;
}

void GlobalQTReceiver::FillMetaStructs()
{
	const char objectName[] = "QTUtils::Internal::GlobalQTReceiver";
	mMetaString.clear();
	mMetaString.insert(mMetaString.begin(), objectName, objectName+sizeof(objectName));
	mMetaString.push_back('\0');

	  /* content:
	  6,       // revision
		0,       // classname
		0,    0, // classinfo
		7,   14, // methods
		0,    0, // properties
		0,    0, // enums/sets
		0,    0, // constructors
		0,       // flags
		0,       // signalCount
		*/
	mMetaData.clear();
	mMetaData.push_back( 6 ); // revision
	mMetaData.push_back( 0 ); // classname
	mMetaData.push_back( 0 ); mMetaData.push_back( 0 ); // classinfo
	mMetaData.push_back( mFuncs.size() ); mMetaData.push_back( 14 ); // methods
	mMetaData.push_back( 0 ); mMetaData.push_back( 0 ); // properties
	mMetaData.push_back( 0 ); mMetaData.push_back( 0 ); // enum/sets
	mMetaData.push_back( 0 ); mMetaData.push_back( 0 ); // constructors
	mMetaData.push_back( 0 ); // flags
	mMetaData.push_back( 0 ); // signalCount

	// slots: signature, parameters, type, tag, flags
	int offset = sizeof(objectName)+1;
	for (auto it = mFuncs.begin(), end = mFuncs.end(); it != end; ++it)
	{
		size_t funcNameLength = it->name.size();
		const char* funcName = it->name.c_str();
		mMetaString.insert(mMetaString.end(), funcName, funcName+funcNameLength);
		mMetaString.push_back('\0');

		mMetaData.push_back( offset ); // signature
		mMetaData.push_back( 36 ); // parameters
		mMetaData.push_back( 36 ); // type
		mMetaData.push_back( 36 ); // tag
		mMetaData.push_back( 0x09 ); // flags
		offset += it->name.size()+1;
	}

	// eod
	mMetaData.push_back( 0 ); 

	mMetaObject.d.superdata  = &QObject::staticMetaObject;
	mMetaObject.d.stringdata = &mMetaString[0];
	mMetaObject.d.data       = &mMetaData[0];
	mMetaObject.d.extradata  = nullptr;
}

```

The idea here is that we maintain the QMetaObject needed to dispatch the call, when qt_metacall is called, to the proper anonymous functions. The magic is done in FillMetaStructs() to maintain the meta object structure each time a new connection is made when `Call(...)` is called.