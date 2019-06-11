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
In the previous article, we [introduced](cpp-development/2019/06/04/templated-properties#generic-properties) the `TSimpleProperty` template class deriving from `TProperty`. It is one of the most important building blocks for declaring properties the way we've seen.

During the writing of this article, I was testing and improving my codebase for declaring automatic commands. In particular, I was playing with tuples and variadic properties when I stumbled upon the C++17 [`std::apply`](https://en.cppreference.com/w/cpp/utility/apply).

## PropertyCommand class

Let's start by introducing the `PropertyCommand` class:

```cpp
template<typename PropType, typename... Values>
class PropertyCommand : public ICommand
{
	typedef typename first<Values...>::type PropValue;

	std::shared_ptr<IPropertyManager> m_Manager;
	std::tuple<Values...> m_InputValues;
	PropValue m_NewValue, m_OldValue;
public:
	PropertyCommand(std::shared_ptr<IPropertyManager> manager, Values... moreValues)
		: m_Manager(manager)
		, m_InputValues(moreValues...)
		, m_NewValue(std::get<0>(m_InputValues))
		, m_OldValue(PropValue())
	{ }
	static std::shared_ptr<ICommand> Create(std::shared_ptr<IPropertyManager> manager, Values... moreValues) {
		return std::static_pointer_cast<ICommand>(std::make_shared<PropertyCommand>(manager, moreValues...));
	}
private:
	void Execute() override
	{
		auto entProperty = m_Manager->GetProperty<PropType>();
		m_OldValue = entProperty->Get();
		std::apply([&](Values... v) {entProperty->Set(v...); }, m_InputValues);
	}

	void Undo() override
	{
		std::tuple<Values...> undoValues = m_InputValues;
		std::get<0>(undoValues) = m_OldValue;

		auto entProperty = m_Manager->GetProperty<PropType>();
		std::apply([&](Values... v) {entProperty->Set(v...); }, undoValues);
	}

	void Redo() override
	{
		auto entProperty = m_Manager->GetProperty<PropType>();
		std::apply([&](Values... v) {entProperty->Set(v...); }, m_InputValues);
	}
};
```