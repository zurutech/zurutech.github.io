---
author: "mmischitelli"
layout: post
did: "mmischi2"
title:  "Automatic commands for Properties"
slug:  Automatic commands for properties
date:   2019-06-06 8:00:00
categories: cpp-development
img: mmischitelli/class-properties/banner.jpg
banner: mmischitelli/class-properties/banner.jpg
tags: cad-bim
description: "Automatic commands for Properties"
---
Using the same concepts illustrated above, using variadic templates and macros we are able to create the skeleton that all the properties will rely on for implementing their commands. Let's start by introducing the `PropertyCommand` base class:

```cpp
template<typename PropType, typename ValueType, typename... AdditionalTypes>
class PropertyCommand : public ICommand
{
    std::shared_ptr<IPropertyManager> m_Manager;
    std::tuple<AdditionalTypes...> m_AdditionalTypes;
    ValueType m_NewValue, m_OldValue;
public:
    PropertyCommand(std::shared_ptr<IPropertyManager> manager, ValueType value, AdditionalTypes... moreValues)
        : m_Manager(manager)
        , m_AdditionalTypes(moreValues)
        , m_NewValue(value)
        , m_OldValue(ValueType())
    { }
    static std::shared_ptr<ICommand> Create(std::shared_ptr<IPropertyManager> manager, ValueType value, AdditionalTypes... moreValues) {
        return std::static_pointer_cast<ICommand>(std::make_shared<SetSizeX>(manager, value, moreValues...));
    }
private:
    void Execute() override;
    void Undo() override;
    void Redo() override;
};

template<typename PropType, typename ValueType, typename... AdditionalTypes>
bool PropertyCommand<PropType, ValueType, AdditionalTypes...>::Execute()
{
    auto entProperty = m_Manager->GetProperty<PropertyType>();
    m_OldValue = entProperty->Get();

    TDelayedVariadicFunction<ValueType, AdditionalTypes...> setter([&](ValueType v, AdditionalTypes... t) {entProperty->Set(v, t...); });
    setter.Execute(m_NewValue, m_AdditionalValues);

    bool result = true;
    if (CheckedExecution::value && !kEntity->VerifyConstraints())
    {
        setter.Execute(m_OldValue, m_AdditionalValues);
        result = false;
    }
}
```