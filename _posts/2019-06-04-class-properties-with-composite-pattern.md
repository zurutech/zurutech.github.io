---
author: "mmischitelli"
layout: post
did: "mmischi1"
title:  "Class properties with the composite pattern"
slug:  Class properties with composite pattern
date:   2019-06-04 8:00:00
categories: cpp-development
img: mmischitelli/class-properties/banner.jpg
banner: mmischitelli/class-properties/banner.jpg
tags: cad-bim
description: "Class properties with the composite pattern"
---
When you usually declare a C++ class, you'd hide its data members declaring them as private. Then you'd also create getters and setter for those members, thus promoting encapsulation. Regardless of where the storage of these properties actually resides, we'd still like to refer them as something that characterizes our class. Something that can be updated, read or even *reverted* to the previous value if something goes wrong or the user presses CTRL+Z.

Using the **command** pattern we should be able to achieve our goal, producing thousands of lines of code sharing a similar structure as a byproduct. I'll first illustrate the pattern, then I'll show how much code those patterns produce. And finally will propose my solution, based upon variadic templates and preprocessor macros, that will cut down the total amount of lines of code for each property from **hundreds** to just **tens**.

## Command Pattern
Most well-designed software and even games often use this pattern. It's basically a way for converting a method call into an object which can be parametrized, put in a queue, logged and makes it easy to implement undoable operations. It's an *an object-oriented replacement for callbacks* (cit. Gang of Four).

The implementation we'll base our discussion on in the following chapters relies on two classes: the **CommandManager** and the **CommandInterface**. The CommandManager is used to *execute*, *undo* or *redo* operations wrapped inside objects that implement the CommandInterface.

```cpp
class MyCommandManager;

class ICommand
{
    friend class MyCommandManager;
public:
    virtual ~ICommand() = default;
private:
    virtual void Execute() = 0;
    virtual void Undo() = 0;
    virtual void Redo() = 0;
};

class MyCommandManager
{
private:    
    std::stack<std::shared_ptr<ICommand>> m_UndoList;
    std::stack<std::shared_ptr<ICommand>> m_RedoList;
public:
    bool Execute(std::shared_ptr<ICommand> command);
    void Undo();
    void Redo();
};
```

Now let's show a concrete example of how commands interact with our objects, how they are created and executed. Imagine we had a `MyTable` class with two properties:

```cpp
class MyTable
{
    float m_SizeX, m_SizeY;
public:
    void SetSizeX(float v) { m_SizeX = v; }
    float GetSizeX() const { return m_SizeX; }
    void SetSizeY(float v) { m_SizeY = v; }
    float GetSizeY() const { return m_SizeY; }
};
```

We'd like to bind those properties to some kind of UI so that the user might be able to modify them directly and undo if they input a wrong number. It all starts with the definition of a command:

```cpp
namespace TableCommands 
{
    class SetSizeX : public ICommand
    {
        std::shared_ptr<MyTable> m_TablePtr;
        float m_NewValue, m_OldValue;
    public:
        SetSizeX(std::shared_ptr<MyTable> table, float newValue)
            : m_TablePtr(table)
            , m_NewValue(newValue)
            , m_OldValue(.0f)
        { }
        static std::shared_ptr<ICommand> Create(std::shared_ptr<MyTable> table, float newValue) {
            return std::static_pointer_cast<ICommand>(std::make_shared<SetSizeX>(table, newValue));
        }
    private:
        void Execute() override {
            m_OldValue = table->GetSizeX();
            table->SetSizeX(m_NewValue);
        }
        void Undo() override {
            table->SetSizeX(m_OldValue);
        }
        void Redo() override {
            table->SetSizeX(m_NewValue);
        }
    };
}
```

The above command was put in its own namespace to avoid naming conflicts. Its `Execute`, `Undo` and `Redo` methods are private and can only be executed by the CommandManager since it was declared as *friend*. We also see an handy static `Create` method that instantiates the command and returns it, wrapped in a generic `ICommand` shared pointer.

What the class really does is self-explanatory: when executed, it stores the old value and sets the new one. Undo and Redo methods just swap the old and the new values accordingly.

Executing this command is straightforward:

```cpp
commandManagerPtr->Execute(TableCommands::SetSizeX::Create(tablePtr, 20.0f));
```

However, we've just created one command for setting the X size. Doing the same for the Y size, as you can easily imagine at this point, would require us to **duplicate** those 27 lines of code and refactor every *SetSizeX* occurrence with *SetSizeY*: another 27 lines of code just to change 1 letter!

## Proposed solution
In order to be able to create commands *automatically* we first need a generic and standard way for defining what a property is and introduce a simple way for accessing them.

### Generic properties
```cpp
typedef void* PropertyID;

template<typename OwnerType>
struct TProperty
{
	virtual ~TProperty() = default;
	OwnerType* Owner;
	bool IsA(PropertyID otherType) const { return _GetPropertyID() == otherType; }
	PropertyID GetPropertyID() const { return _GetPropertyID(); }
private:
	virtual PropertyID _GetPropertyID() const = 0;
};

template<typename OwnerType, typename PropertyType, typename... AdditionalTypes>
struct TSimpleProperty : TProperty<OwnerType>
{
	void Set(PropertyType value, AdditionalTypes... moreValues) { 
        _Set(value, moreValues...); 
    }
	PropertyType Get() const { return _Get(); }
	std::shared_ptr<ICommand> Command(PropertyType value, AdditionalTypes... moreValues) const {
        return _Command(value, moreValues...); 
    }

private:
	virtual void _Set(PropertyType value, AdditionalTypes... moreValues) = 0;
	virtual PropertyType _Get() const = 0;
	virtual std::shared_ptr<ICommand> _Command(PropertyType value, AdditionalTypes... moreValues) const = 0;
	PropertyID _GetPropertyID() const override = 0;
};
```

Both `TProperty` and `TSimpleProperty` are abstract structs and cannot be instantiated. They serve different purposes: `TProperty` is the most basic interface that is shared by any kind of property we'd like to have; `TSimpleProperty` is one of these different types (the simplest one).

Please note that `TSimpleProperty` is still quite complex: it is a *variadic template* and is characterized by a variable amount of types. These additional types are used (and passed) to the Set and Command methods and are handy for developers wanting to execute some additional logic during a Set, based on the provided parameters. In my opinion it's better to keep setters as simple as possible and instead move whatever additional logic in a separate method. However, in the real world, if a developer sees that some logic needs always to be executed after setting some particular property, he'd certainly embed it in the setter.

At this point we can then proceed to create the **SizeX** property in our `MyTable` class:
```cpp
class MyTable
{
#pragma region SizeX property
public:
    friend struct TSizeXProperty;
    struct TSizeXProperty : TSimpleProperty<MyTable,float> { 
    public: 
        static PropertyID GetStaticPropertyID() { 
            return PropertyID(&TSizeXProperty::s_PropID); 
        } 
    private:
        static const TCHAR s_PropID = 1;
        PropertyID _GetPropertyID() const override { 
            return GetStaticPropertyID(); 
        }
        void _Set(float value) override;
        float _Get() const override;
        std::shared_ptr<ICommand> _Command(float value) const override;
    }; 
    void SetSizeX(float value) { m_SizeXProperty.Set(value); } 
    float GetSizeX() const { return m_SizeXProperty.Get(); } 
private:
    TSizeXProperty m_SizeXProperty;
#pragma endregion

    void Init(){
        m_SizeXProperty.Owner = this;
    }
};

void MyTable::TSizeXProperty::_Set(float value) { Owner->... }
float MyTable::TSizeXProperty::_Get() const { return Owner->... }
std::shared_ptr<ICommand> MyTable::TSizeXProperty::_Command(float value) const override {
    return ...
}
```

In the above code, we first declare that our `TSizeXProperty` struct is a friend of `MyTable`: this will allow the code executed in the private _Set and _Get method to have full access to private methods in the Table class.

`TSizeXProperty` derives from `TSimpleProperty` and as such must override the pure virtual methods `_Get`, `_Set`, `_Command` and `_GetPropertyId`. The first three are quite obvious at this point while the last one is a trick we'll use next to perform some kind of *type checking* at runtime without resorting to RTTI: we're returning the *pointer* to the static `s_PropID` and comparing it against other properties. All instances of the same property class will thus share the same memory location for that private variable, resulting in some form of type checking that's useful to our goal.

After the declaration of the property's struct, we also declared and defined Setters and Getters for that property directly in the `MyTable` class. Those methods are just proxies to the same methods defined in the property's struct and allow interacting with it without knowing how that property was implemented.

Lastly, we need to initialize our property by setting the owner pointer to the actual `MyTable`'s object.

### Generic properties with macros
All this variadic and nested structs is nice and all but we still need to write a lot of code (and we haven't even created our commands yet!). Right now, however, we have all we need to transform the above Property struct code into just one single line of code:

```cpp
class MyTable
{
    DECLARE_PROPERTY(MyTable, float, SizeX)
    DECLARE_PROPERTY(MyTable, float, SizeY)

    void Init(){
        m_SizeXProperty.Owner = this;
        m_SizeYProperty.Owner = this;
    }
};
```

What we've done here is putting all the code enclosed in the `#pragma region` in a preprocessor macro, replacing types and names with the arguments. Since the underlying structs support a variable amount of parameters, we also needed to create some more macros:

```cpp
DECLARE_PROPERTY(ClassType, Type1, Name1) ...
DECLARE_PROPERTY_2P(ClassType, Type1, Name1, Type2, Name2) ...
DECLARE_PROPERTY_3P(ClassType, Type1, Name1, Type2, Name2, Type3, Name3) ...
```

### Accessing generic properties in a generic way
Right now, even with the above macros, we have just introduced a quick way of creating properties: we still need to find a way to allow developers to access these properties by just indicating their *type*. That would completely change how we can interact with our class and pave the way for *automatic commands*.

Fortunately, most of work's been already done! Let's now introduce the `IPropertyManager` interface:

```cpp
class IPropertyManager
{
    typedef std::map<PropertyID, void*> MapType
    MapType m_Properties;
public:
    template<typename PropertyType>
    PropertyType* GetProperty() const
    {
        auto prop = m_Properties.find(PropertyType::GetStaticPropertyId());
        if (prop != m_Properties.end()) {
            return static_cast<PropertyType*>(prop->second);
        }
        return nullptr;
    }
protected:
    template<typename ClassType>
    void _InitProperty(TEntityProperty<ClassType>& prop)
    {
        prop.Owner = static_cast<ClassType*>(this);
        m_Properties.insert(std::pair<PropertyID, void*>(prop.GetPropertyID(), &prop));
    }
};
```

We can thus modify the `MyTable` class as follows:

```cpp
class MyTable : public IPropertyManager
{
    DECLARE_PROPERTY(MyTable, float, SizeX)
    DECLARE_PROPERTY(MyTable, float, SizeY)

    void Init(){
        _InitProperty(m_SizeXProperty);
        _InitProperty(m_SizeYProperty);
    }
};
```

And we're able to interact with properties like this:

```cpp
tablePtr->GetProperty<MyTable::TSizeXProperty>()->Set(3.0f);
const float kSizeY = tablePtr->GetProperty<MyTable::TSizeXProperty>()->Get();
```