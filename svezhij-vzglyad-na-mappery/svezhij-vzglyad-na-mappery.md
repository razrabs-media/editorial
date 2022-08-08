# Свежий взгляд на мапперы

![img](preview.png)

Если вас, как и меня, не устраивает AutoMapper и его аналоги, приглашаю обсудить идеи для нового маппера, которого мы заслуживаем

## Дисклеймер

Я не претендую на истину, а просто хочу поделиться своими мыслями. Я использую в примерах [AutoMapper](https://automapper.org/) потому что он самый популярный, я только его в своей работе и использовал.

## Подключение

Начну с подключения маппера к проекту. Чтобы начать пользоваться автомаппером, необходимо создать конфигурацию, указать в ней, какие классы мы хотим мапить, и создать маппер на основе этой конфигурации. Мне бы хотелось избавиться от лишних действий, почему недостаточно одного лишь вызова метода для маппинга? В нем есть исходный объект и его тип, а также можно указать тип объекта, являющегося целью для маппинга.

Так это могло бы выглядеть. Никаких конфигураций, просто метод Map из которого видно, что я хочу смапить объект source в объект типа UserDestination.
```C#
using System;
using NextGenMapper;

namespace NextGenMapperDemo
{
    class Program
    {
        static void Main()
        {
            var source = new UserSource("Vasya", "Pupkin", new DateTime(2007, 01, 01));

            var destination = source.Map<UserDestination>();

            Console.WriteLine(destination.ToString());
        }
    }

    public record UserSource(string FirstName, string SecondName, DateTime Birthday);
    public record UserDestination(string FirstName, string SecondName, DateTime Birthday);
}
```

## Лишняя конфигурация

Допустим, наш тип не совсем простой, одно из его свойств это не примитивный тип, а тип, определённый нами. Автомапппер не станет мапить такое свойство, он будет ругаться, что конфигурации для маппинга этого типа не найдено.

```C#
internal class Program
{
    static void Main(string[] args)
    {
        var config = new MapperConfiguration(cfg => cfg.CreateMap<Source, Destination>());
        var mapper = config.CreateMapper();

        var source = new Source { Property = new SourceProperty { Value = 123 } };

        var destination = mapper.Map<Source, Destination>(source);
    }
}

public class Source
{
    public SourceProperty Property { get; set; }
};
public class Destination
{
    public DestinationProperty Property { get; set; }
};
public class SourceProperty
{
    public int Value { get; set; }
}
public class DestinationProperty
{
    public int Value { get; set; }
}
```

Говоря о примере выше, почему бы не попробовать смапить `SourceProperty` на `DestinationProperty`? Я хочу смапить `Source` на `Destination`, разумно предположить, что это какие-то схожие типы, что свойства с одинаковыми именами должны мапиться. Мало того что я должен добавить в конфигурацию классы, которые хочу смаппить, я также должен добавить в конфигурацию все классы свойств.

## Совсем без конфигурации нельзя

Всё было бы куда проще, если типы для маппинга всегда были бы идентичны, свойства с одинаковыми именами, не отличающиеся по типу. Но мы живем в реальности, и нам необходима возможность указать мапперу, как именно он должен мапить определённые типы.

Представим что у нас есть два типа, для правильного маппинга одного в другой нам необходимо объяснить, что `FirstName` и `LastName` превращаются в `Name`, а `Login` в `Email`. `Age` должен смаппиться автоматически.
```C#
public class UserSource
{
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public string Login { get; set; }
    public int Age { get; set; }
}

public class UserDestination
{
    public string Name { get; set; }
    public string Email { get; set; }
    public int Age { get; set; }
}
```

Для автомаппера задача решается вызовом специального апи для конфигурации. Одна из проблем данного подхода, это необходимость учить эти самые методы для описания конфигурации, которых может быть довольно много. Такой код сложно воспринимать, по сравнению с кодом, который мы написали бы вручную для такого маппинга.
```C#
var config = new MapperConfiguration(cfg => cfg.CreateMap<UserSource, UserDestination>()
    .ForMember("Name", opt => opt.MapFrom(src => src.FirstName + " " + src.LastName))
    .ForMember("Email", opt => opt.MapFrom(src => src.Login)));
```

Можно немного улучшить способ, показанный выше, добавив название свойства в имена методов. Выглядит немного проще и понятнее, и на этом отличия заканчиваются.
```C#
var config = new MapperConfiguration(cfg => cfg.CreateMap<UserSource, UserDestination>()
    .NameFrom(src => src.FirstName + " " + src.LastName)
    .EmailFrom(src => src.Login));
```

А вот более радикальный, и как мне кажется, более предпочтительный способ. Мы пишем код, похожий на маппинг вручную, только вместо того, чтобы вызывать конструктор класса, мы вызываем специальный метод маппера, куда передаем определённые нами свойства. Такой подход позволяет не выдумывать новый апи.
```C#
UserDestination Map(this UserSource src) => src.MapWith(
    fullName: src.FirstName + " " + src.LastName, 
    email: src.Login
);
```

## Немного о внутренностях

Как бы хорошо мы не выучили апи автомаппера, рано или поздно мы усомнимся в правильности конфига который мы написали, и у нас встанет вопрос: а что там внутри, собственно, происходит? Мы написали конфиг, вроде он работает, но данные на выходе не совсем те, что мы ожидаем, как же нам узнать, где именно ошибка? Или немного другая ситуация, компиляция проходит, но когда приложение запущено, при попытке смаппить объекты падает ошибка, а вместо понятного сообщения у нас пелена непонятного текста. 
```
[NullReferenceException: Object reference not set to an instance of an object.] AutoMapper.Mappers.TypeMapMapper.Map(ResolutionContext context, IMappingEngineRunner mapper) +116
AutoMapper.MappingEngine.AutoMapper.IMappingEngineRunner.Map(ResolutionContext context) +459

[AutoMapperMappingException:

Mapping types: String -> String System.String -> System.String

Destination path: CodeModel.Code

Source value: E]
AutoMapper.MappingEngine.AutoMapper.IMappingEngineRunner.Map(ResolutionContext context) +537
AutoMapper.Mappers.DataReaderMapper.MapPropertyValue(ResolutionContext context, IMappingEngineRunner mapper, Object mappedObject, PropertyMap propertyMap) +305
AutoMapper.Mappers.DataReaderMapper.MapPropertyValues(ResolutionContext context, IMappingEngineRunner mapper, Object result) +210
AutoMapper.Mappers.DataReaderMapper.Map(ResolutionContext context, IMappingEngineRunner mapper) +639
AutoMapper.MappingEngine.AutoMapper.IMappingEngineRunner.Map(ResolutionContext context) +477 AutoMapper.MappingEngine.Map(Object source, Type sourceType, Type destinationType, Action1 opts) +176

  AutoMapper.MappingEngine.Map(Object source, Action1 opts) +162
AutoMapper.MappingEngine.Map(Object source) +143
```

И для этой задачи у автомаппера найдется решение. Мы можем вызвать у конфигурации метод `BuildExecutionPlan`, который вернет нам дерево выражений, построенное автомаппером. Чтобы читать его было проще, нам предлагают использовать расширение для студии. Наверно, никто не будет спорить с тем, что нажать F12 (Go To Definition) на интересующем нас методе Map и посмотреть код который будет вызван, намного проще, а главное привычнее. 

Вот какой код мы должны увидеть, простой и понятный, такой мы бы написали сами. Нужно ли говорить, что, попробовав посмотреть код метода Map автомаппера, картина будет совсем другой.
```C#
public static UserDestination Map(this UserSource source) 
    => new UserDestination
    (
        source.FirstName,
        source.SecondName,
        source.Birthday
    );
```

## Ошибки времени компиляции

Я считаю, и думаю многие со мной согласятся, что в идеальном маппере все ошибки пользователя должны быть выявлены во время компиляции. Маппер должен следить за тем, что он сконфигурирован валидно и используется правильно, в ином случае, проект не может быть сбилжен. Не уверен, что это возможно на 100%, но к этому нужно стремиться.

Пример рантайм ошибки которую следовало бы избежать. В конструктор класса `Destination` необходимо передать два аргумента, и маппер не знает где взять второй. Этот код спокойно билдится, но когда мы его запускаем и доходим до строчки `var destination = mapper.Map<MySource, Destination>(source);` - падает исключение.
```C#
internal class Program
{
    static void Main(string[] args)
    {
        var config = new MapperConfiguration(cfg => cfg.CreateMap<Source, Destination>());
        var mapper = config.CreateMapper();

        var source = new Source("123");

        var destination = mapper.Map<Source, Destination>(source);
    }
}

public class Source
{
    public string Property { get; set; }
};
public class Destination
{
    public string Property { get; }
    public string AnotherProperty { get; }

    public Destination(string property, string anotherProperty)
    {
        Property = property;
        AnotherProperty = anotherProperty;
    }
};
```

## Производительность

Разумеется, хочется чтобы автоматический маппер работал так же быстро, как методы написанные вручную. Чтобы он использовал минимум оперативной памяти. Чтобы он не увеличивал время старта приложения, и время билда.

## Реальность или вымысел?

На первый взгляд приведённые примеры кода выглядят невозможными. Их можно написать руками для каждой отдельной пары типов, но никак не универсальный вариант. На самом деле это всё стало реальным после появления технологии [C# Source Generators](https://docs.microsoft.com/en-us/dotnet/csharp/roslyn-sdk/source-generators-overview). Эта технология позволяет проанализировать код, написанный программистом, сгенерировать на основе этого анализа дополнительный код и добавить его в сборку. Эти идеи я пытаюсь воплотить в своем маппере ([ссылка на github](https://github.com/DedAnton/NextGenMapper)). 

Буду рад услышать конструктивную критику, а также ваше мнение о том, как должен выглядеть идеальный маппер.