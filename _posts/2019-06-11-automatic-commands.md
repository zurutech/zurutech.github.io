---
author: "mmischitelli"
layout: post
did: "mmischi2"
title:  "Automatic commands for class properties"
slug:  Automatic commands for class properties
date:   2019-06-11 8:00:00
categories: cpp-development
img: mmischitelli/class-properties/autocommands.jpg
banner: mmischitelli/class-properties/autocommands.jpg
tags: coding
description: "Using class properties to automatically generate commands"
---
In the [previous post](/cpp-development/2019/06/04/templated-properties), we've seen how to use variadic templates to declare generic *class properties*. Each so-called *property* is just a struct that declares a setter and a getter and their implementation is left to the developer. Adopting this convention, we're able to make these properties *queryable*: developers can query objects implementing the `IPropertyManager` interface for specific properties, without knowing the object's class nor if it actually has that property.

Using these properties and adopting the same concepts, we're going to introduce a generic [command](/cpp-development/2019/06/04/templated-properties#command-pattern) that will be used to automatically make write operations on properties *undoable*.

## Revised SimpleProperty template
In the previous article, I've [introduced](/cpp-development/2019/06/04/templated-properties#generic-properties) the `TSimpleProperty` template class deriving from `TProperty`. It is one of the most important building blocks for declaring properties the way we've seen.

During the writing of this article, I was testing and improving my codebase for declaring automatic commands. In particular, I was playing with tuples and variadic properties when I stumbled upon the C++17 [`std::apply`](https://en.cppreference.com/w/cpp/utility/apply): it allows calling a *callable object* (in our case, a variadic method) with a tuple of arguments.

This new possibility triggered my creativity and, after some researching and experimenting, made me reconsider the need of having `PropertyType` separated from `AdditionalTypes` in the `TSimpleProperty` templated class. The main reason I was doing this was due to being able to declare the return type in the getter method of any given property.

However, with a little modification, we can get rid of this and put all the types together as variadic arguments in the `TSimpleType` declaration:

```cpp
template <typename T1, typename ...T>
struct first { typedef T1 type; };

template<typename OwnerType, typename... Values>
struct TSimpleProperty : TProperty<OwnerType>
{
	typedef typename first<Values...>::type PropertyType;

	void Set(Values... values) {
		_Set(values...);
	}
	PropertyType Get() const { return _Get(); }
	std::shared_ptr<ICommand> Command(Values... values) const {
		return _Command(values...);
	}

private:
	virtual void _Set(Values... values) = 0;
	virtual PropertyType _Get() const = 0;
	virtual std::shared_ptr<ICommand> _Command(Values... values) const = 0;
	PropertyID _GetPropertyID() const override = 0;
};
```

This simplification was made possible thanks to the *templated magic* contained in the `first` struct declared at the top of the above code.

When we declare the `first` struct using the variadic arguments, they're split into the *first* and *all the others* types. The former is then put in the typedef declared in the struct so that we can later access it, when we *further* typedef it in our Property as being its main `PropertyType`. That's it! The need to divide the first type from the others no longer exists.

Please also note that the MACRO we used in the previous article to speed up the creation of properties should be updated to reflect these changes.

## PropertyCommand class

Back to the topic of this article: automatic commands! Let's start by introducing the `PropertyCommand` class:

```cpp
template<typename Property, typename... Values>
class PropertyCommand : public ICommand
{
	typedef typename first<Values...>::type PropValue;

	Property* m_Property;
	std::tuple<Values...> m_InputValues;
	std::tuple<Values...> m_UndoValues;
public:
	PropertyCommand(std::shared_ptr<IPropertyManager> manager, Values... inputValues)
		: m_InputValues(inputValues...)
	{
		m_Property = manager->GetProperty<Property>();
		auto oldValue = m_Property->Get();

		m_UndoValues = m_InputValues;
		std::get<0>(m_UndoValues) = oldValue;
	}
	static std::shared_ptr<ICommand> Create(std::shared_ptr<IPropertyManager> manager, Values... inputValues) {
		return std::static_pointer_cast<ICommand>(std::make_shared<PropertyCommand>(manager, inputValues...));
	}
private:
	void Execute() override {
		std::apply([&](Values... v) {m_Property->Set(v...); }, m_InputValues);
	}
	void Undo() override {
		std::apply([&](Values... v) {m_Property->Set(v...); }, m_UndoValues);
	}
	void Redo() override {
		std::apply([&](Values... v) {m_Property->Set(v...); }, m_InputValues);
	}
};
```

As you can see, simplifying our properties also makes the above code almost trivial. The class' architecture is almost identical to the ad-hoc implementation we wrote in the previous post for `TableCommands::SetSizeX`. The most important differences are in the constructor, where we store the variadic arguments in a `std::tuple`, read the current property's value and put it in another tuple that will be later used in the `Undo` method. Also the `Execute`, `Undo`, `Redo` methods have been hugely simplified thanks to `std::apply`, giving us the possibility of invoking the property's setter with a specific tuple of arguments.

## Declaring Automatic Properties
With the above template, along with the revised property structure, we have all that's needed to declare commands in **just one line of code** (as promised!):

```cpp
namespace TableCommands
{
    class SetSizeXCmd : public PropertyCommand<MyTable::TSizeXProperty,float> { };
    class SetSizeYCmd : public PropertyCommand<MyTable::TSizeYProperty,float> { };
}
```

## Testing
Let's see how we can put all we've seen together in a very simple test program:

```cpp
int main()
{
	auto cmdMgr = std::make_shared<MyCommandManager>();
	auto tablePtr = std::make_shared<MyTable>();
	tablePtr->Init();

	tablePtr->GetProperty<MyTable::TSizeXProperty>()->Set(10.0f);
	std::cout << "[Start] SizeX: " << tablePtr->GetSizeX() << "\n";

	cmdMgr->Execute(TableCommands::SetSizeXCmd::Create(tablePtr, 20.0f));
	std::cout << "[Execute] SizeX: " << tablePtr->GetSizeX() << "\n";

	cmdMgr->Undo();
	std::cout << "[Undo] SizeX: " << tablePtr->GetSizeX() << "\n";

	cmdMgr->Redo();
	std::cout << "[Redo] SizeX: " << tablePtr->GetSizeX() << "\n";
}

===========================
Output:

[Start] SizeX: 10
[Execute] SizeX: 20
[Undo] SizeX: 10
[Redo] SizeX: 20
```

In the minimal code above, we first create instances of our command manager and the `MyTable` test class. Then we set the table's X-size to `10` through its property (alternatively, we could have simply called `->SetSizeX()`) and print its value just to make sure everything works. Then we play a bit with the newly created `SetSizeXCmd` command, updating and reverting the table's X-size, printing it each time to track every change.

## Conclusions
In these two articles, I proposed a solid property system that can be used to normalise how developers can get or set values, revert updates and even query classes for some specific *property*.

I decided to have more code in the declaration of properties instead of having even more code for commands. In my opinion, if your main use of commands is just to undo/redo values, then there is no need to have complexity in them and they should be as simple as they can be. This will prevent developers from putting custom logic in the command, hiding under the carpet the *dirt* of your codebase.