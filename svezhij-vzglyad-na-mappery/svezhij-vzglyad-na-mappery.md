# Свежий взгляд на мапперы

![img](preview.png)

Если вас, как и меня, не устраивает AutoMapper и его аналоги, приглашаю обсудить идеи для нового маппера, которого мы заслуживаем.

## Что такое маппинг и зачем он нужен
Проще всего объяснить на примере, представим что у нас есть база данных с табличкой Users, у которого есть поля Id, Name, Email, CreatedAt, IsDeleted. Мы хотим написать апи, который позволит по Id получить пользователя. Но нам не нужны все поля, хранящиеся в базе, нас интересуют только Name и Email, остальные - это технические детали.
Итак, у нас есть два класса с разным набором полей, представляющие пользователя. В некоторых языках никакой проблемы вообще не возникнет, например в TypeScript.

```typescript
type UserDb = {
    id: number,
    name: string,
    email: string,
    createdAt: string,
    isDeleted: boolean
};

type UserApi = {
    name: string,
    email: string
}

function printUser(user: UserApi) {
    console.log(user);
}

let userDb : UserDb = {
    id: 1,
    name: 'Anton',
    email: 'Ryabchikov',
    createdAt: '12.09.2022',
    isDeleted: false
};

printUser(userDb); 
```
Это валидный код, и он будет делать именно то, что мы от него ожидаем. Компилятор сам понимает, что у UserDb есть поля name и email, поэтому может принять объект этого типа в метод printUser. Если мы изменим имя поля name в UserDb, то компилятор скажет нам об ошибке: "Argument of type 'UserDb' is not assignable to parameter of type 'UserApi'. Property 'name' is missing in type 'UserDb' but required in type 'UserApi'". В C# Подобное невозможно из-за более строгой системы типов. Если у нас есть тип A и тип B, одинаковые у них поля или нет, для компилятора это будут разные типы, несовместимые друг с другом.

```C#
var userDb = new UserDb
{
    Id = 1,
    Name = "Anton",
    Email = "Ryabchikov",
    CreatedAt = "12.09.2022",
    IsDeleted = false
};

Test.PrintUser(userDb);

class UserDb
{
    public int Id { get; set; }
    public string Name { get; set; }
    public string Email { get; set; }
    public string CreatedAt { get; set; }
    public bool IsDeleted { get; set; }
}

class UserApi
{
    public string Name { get; set; }
    public string Email { get; set; }
}

static class Test
{
    public static void PrintUser(UserApi user)
    {
        Console.WriteLine($"name: {user.Name}, email: {user.Email}");
    }
}
```
Поэтому аналогичный код в C# не будет работать, компилятор выдаст ошибку: "cannot convert from 'UserDb' to 'UserApi'"

Чтобы заставить его работать, нам нужно написать метод для маппинга UserDb в UserApi.

```C#
var userDb = new UserDb
{
    Id = 1,
    Name = "Anton",
    Email = "Ryabchikov",
    CreatedAt = "12.09.2022",
    IsDeleted = false
};

var mappedUser = Mapper.Map(userDb);

Test.PrintUser(mappedUser);

class UserDb
{
    public int Id { get; set; }
    public string Name { get; set; }
    public string Email { get; set; }
    public string CreatedAt { get; set; }
    public bool IsDeleted { get; set; }
}

class UserApi
{
    public string Name { get; set; }
    public string Email { get; set; }
}

static class Test
{
    public static void PrintUser(UserApi user)
    {
        Console.WriteLine($"name: {user.Name}, email: {user.Email}");
    }
}

static class Mapper
{
    public static UserApi Map(UserDb user)
    {
        return new UserApi
        {
            Name = user.Name,
            Email = user.Email
        };
    }
}
```
Теперь всё заработает.

Может показаться, что написать метод для маппинга не так уж и сложно, но проблема в том, что даже в небольшом проекте, требуется написать десятки, иногда сотни таких методов. Это рутинная работа, от которой мало кто может получать удовольствие. Поэтому мы используем AutoMapper и его аналоги, он позваляет сильно сократить количество написанного кода, избавляя нас от головной боли, особенно в простых случаях, когда модели не сильно отличаются друг от друга.

Но AutoMapper не предел мечтаний, у него есть свои недостатки, поэтому я расскажу про своё виденье маппера, и о том, как его реализовать.

## Подключение

Начну с подключения маппера к проекту. Чтобы начать пользоваться автомаппером, необходимо создать конфигурацию, указать в ней, какие классы мы хотим маппить, и создать маппер на основе этой конфигурации.
```C#
using AutoMapper;
using System;

var config = new MapperConfiguration(cfg =>
{
    cfg.CreateMap<UserSource, UserDestination>();
});
var mapper = config.CreateMapper();

var source = new UserSource("Vasya", "Pupkin", new DateTime(2007, 01, 01));

var destination = mapper.Map<UserSource, UserDestination>(source);

Console.WriteLine(destination);

public record UserSource(string FirstName, string SecondName, DateTime Birthday);
public record UserDestination(string FirstName, string SecondName, DateTime Birthday);
//record в C# - это тот же class, но с более простым описанием и автоматической реализацией некоторых методов, далее я буду использовать их.
```
 Если вы используете контейнер зависимостей, алгоритм будет другой. Но суть одна - множество действий просто для того чтобы начать пользоваться маппером. Мне бы хотелось избавиться от лишнего. Почему недостаточно одного лишь вызова метода для маппинга? В нем есть исходный объект и его тип, а также можно указать тип объекта, являющегося целью для маппинга.


Так это могло бы выглядеть. Никаких конфигураций, просто метод Map из которого видно, что я хочу смаппить объект source в объект типа UserDestination.
```C#
using NextGenMapper;

var source = new UserSource("Vasya", "Pupkin", new DateTime(2007, 01, 01));

var destination = source.Map<UserDestination>();

Console.WriteLine(destination);

public record UserSource(string FirstName, string SecondName, DateTime Birthday);
public record UserDestination(string FirstName, string SecondName, DateTime Birthday);
```

## Конфигурация

### Свойство не примитивного типа
Допустим, наш класс не совсем простой, одно из его свойств это не примитивный тип, а другой класс, определённый нами. 
```C#
record Source(SourceProperty Property);
record SourceProperty(int Value);
маппить
record Destination(DestinationProperty Property);
record DestinationProperty(int Value);
```
Автомапппер не станет маппить такое свойство, он будет ругаться, что конфигурации для маппинга этого типа не найдено. Поэтому мне придется добавить в конфигурацию не только типы которые я хочу смаппить, но и типы их свойств, если они не являются примитивными.
```C#
using AutoMapper;

var config = new MapperConfiguration(cfg =>
{
    cfg.CreateMap<Source, Destination>();
    cfg.CreateMap<SourceProperty, DestinationProperty>();
});
var mapper = config.CreateMapper();

var source = new Source(new SourceProperty(123));

var destination = mapper.Map<Source, Destination>(source);

Console.WriteLine(destination);
```

Но почему бы не попробовать смаппить `SourceProperty` на `DestinationProperty`? Я хочу смаппить `Source` на `Destination`, разумно предположить, что это какие-то схожие типы, что свойства с одинаковыми именами должны маппиться.
```C#
using NextGenMapper;

var source = new Source(new SourceProperty(123));

var destination = source.Map<UserDestination>();

Console.WriteLine(destination);
```

### Кастомизация маппинга
Всё было бы куда проще, если типы для маппинга всегда были бы идентичны, свойства с одинаковыми именами, не отличающиеся по типу. Но мы живем не в сказке, и нам необходима возможность указать мапперу, как именно он должен маппить определённые типы.

Представим что у нас есть два класса, для правильного маппинга одного в другой нам необходимо указать, что `FirstName` и `LastName` превращаются в `Name`, а `Login` в `Email`. `Age` должен смаппиться автоматически.
```C#
public record UserSource(string FirstName, string LastName, string Login, int Age);
public record UserDestination(string Name, string Email, int Age);
```

Для автомаппера задача решается вызовом специального апи для конфигурации маппинга. Одна из проблем данного подхода, это необходимость учить эти самые методы для описания конфигурации, которых может быть довольно много. Такой код сложнее воспринимать, по сравнению с кодом, который мы написали бы вручную для такого маппинга. Но это всё цветочки, по сравнению с удобством отладки находящимся примерно на уровне нуля. Ты не можешь просто поставить точку останова и посмотреть как маппится определённый объект, всё это скрыто глубоко в недрах автомаппера. Да, с помощью метода BuildExecutionPlan, у нас есть возможность посмотреть дерево выражений, которое будет выполняться для маппинга. Но это не то удобство к которому я привык, хотя признаю, что это лучше чем ничего.
```C#
var config = new MapperConfiguration(cfg => cfg.CreateMap<UserSource, UserDestination>()
    .ForMember("Name", opt => opt.MapFrom(src => src.FirstName + " " + src.LastName))
    .ForMember("Email", opt => opt.MapFrom(src => src.Login)));
```

Я против такого подхода, зачем мне писать код, который описывает как нужно маппить, если я могу написать код, который будет маппить?
```C#
var destination = source.MapWith<UserDestination>(
    name: src.FirstName + " " + src.LastName, 
    email: src.Login);
```
Запомнить нужно только метод MapWith, а для отладки достаточно поставить точку останова. 

Конечно я продемонстрировал не совсем эквивалентный код, ведь в последнем случае придется описывать одно и тоже каждый раз. К тому же маппер, столкнувшись с объектами, у свойств которых типы `UserSource` и `UserDestination`, не поймёт, как их смаппить. Чтобы этого избежать нужен способ указать мапперу, как маппить определённые типы. И, на мой скромный взгляд, лучший вариант - это написать метод, который маппер просто будет использовать.
```C#
namespace NextGenMapper;

internal static partial class Mapper
{
    internal static UserDestination Map<To>(this UserSource source) 
        => source.MapWith<UserDestination>(name: src.FirstName + " " + src.LastName, email: src.Login);
}
```
Теперь этот метод могу использовать я, а также может использовать маппер, при маппинге других типов, например массива `UserSource[]` в `UserDestination[]`.

### Сложный пример
Предлагаю рассмотреть более сложный пример, он умышленно переусложнен, и в реальности такое вряд ли встретиться, но он позволит лучше увидеть разницу
```C#
record UserSource(
    Guid Id,
    string FirstName,
    string LastName,
    string Email,
    DateTime BirthDate,
    bool IsDeleted,
    string Country,
    string State,
    string City,
    string Street,
    UserSettingsSource Settings,
    AuthSource Auth);
record UserSettingsSource(TimeZone TimeZone, Theme Theme, bool IsNotificationsEnabled);
record TimeZone(int Id, string Name);
record Theme(int Id, string Name);
record AuthSource(Guid PrincipleId, string Role);

record UserDestination(
    string Id,
    string Name,
    string Login,
    int Age,
    string BirthDate,
    AddressDestination Address,
    UserSettingsDestination Settings,
    AuthDestination Auth);
record AddressDestination(string Country, string State, string City, string Street);
record UserSettingsDestination(int TimeZoneId, int ThemeId, bool IsNotificationsEnabled);
record AuthDestination(string PrincipleId, string Role);
```

```C#
var config = new MapperConfiguration(cfg =>
{
    cfg.CreateMap<UserSource, UserDestination>()
        .ForMember(dst => dst.Name, opt => opt.MapFrom(src => $"{src.FirstName} {src.LastName}"))
        .ForMember(dst => dst.Login, opt => opt.MapFrom(src => src.Email))
        .ForMember(dst => dst.BirthDate, opt => opt.MapFrom(src => src.BirthDate.ToShortDateString()))
        .ForMember(dst => dst.Age, opt => opt.MapFrom(src => Utils.AgeFromBirthDate(src.BirthDate)))
        .ForMember(dst => dst.Address, opt => opt.MapFrom(src => new AddressDestination(src.Country, src.State, src.City, src.Street)));
    cfg.CreateMap<UserSettingsSource, UserSettingsDestination>()
        .ForMember(dst => dst.ThemeId, opt => opt.MapFrom(src => src.Theme.Id))
        .ForMember(dst => dst.TimeZoneId, opt => opt.MapFrom(src => src.TimeZone.Id));
    cfg.CreateMap<AuthSource, AuthDestination>();
    cfg.CreateMap<Guid, string>().ConvertUsing(x => x.ToString("N"));
});

// Пришлось вынести в отдельный метод, потому что "A lambda expression with a statement body cannot be converted to an expression tree"
static int AgeFromBirthDate(DateTime birthdate)
{
    var today = DateTime.Today;
    var age = today.Year - birthdate.Year;
    if (birthdate.Date > today.AddYears(-age))
    {
        age--;
    };

    return age;
}
```
Пока писал кофигурацию для маппинга, автомаппер повставлял мне немного палок в колеса: заставил вынести логику подсчета возраста в отдельный метод AgeFromBirthDate, и переделать тип `UserDestination` в класс вместо рекорда потому что не смог использовать его конструктор.

Альтернативный подход, в котором настройка выглядит как обычный код, не сильно отличающийся от того, что я мог бы написать для ручного маппинга. Этот код будет вызываться каждый раз при маппинге объектов типа `UserSource -> UserDestination`. 
```C#
using System;

namespace NextGenMapper;

internal static partial class Mapper
{
    internal static string Map<To>(this Guid source) => source.ToString("N");

    internal static UserSettingsDestination Map<To>(this UserSettingsSource source)
        => source.MapWith<UserSettingsDestination>(timeZoneId: source.TimeZone.Id, themeId: source.Theme.Id);

    internal static UserDestination Map<To>(this UserSource source)
    {
        var today = DateTime.Today;
        var age = today.Year - source.BirthDate.Year;
        if (source.BirthDate.Date > today.AddYears(-age))
        {
            age--;
        };
        return source.MapWith<UserDestination>(
            name: $"{source.FirstName} {source.LastName}",
            login: source.Email,
            birthDate: source.BirthDate.ToShortDateString(),
            age: age,
            address: new AddressDestination(source.Country, source.State, source.City, source.Street));
    }
}
```

## Как это работает?
Перед тем как рассказывать о преимуществах моего подхода, стоит рассказать о том, как реализовать то, что я предлагаю, и чем это отличается от автомаппера.

Упрощенно, автомаппер работает следующим образом:
 1. Из конфигурации, которую мы описали создаются деревья выражений, они описывают то, как должен проходить маппинг
 2. Далее эти деревья выражений компилируются
 3. И помещаются в словарь, а ключем для них является пара типов для маппинга 
 4. Когда мы вызываем метод `Map`, мы ищем в словаре нужный нам делегат
 5. И вызываем его, передав в него наш объект в качестве аргумента

Моя же реализация не использует деревья выражений. Я использую генераторы исходного кода (Source Generators), относительно новую технологию, добавленную примерно два года назад. Генераторы исходного кода позволяют проанализировать синтаксис и семантику пользовательского кода, сгенерировать на основе этого новый код, и внедрить его в результат компиляции.
![source generators visualisation schema](source-generator-visualization.png)

Если говорить проще, используя сурс генераторы, я могу посмотреть, какой код написал пользователь, и проанализировав его, сгенерировать новый код, который будет добавлен прямо в сборку к пользователю, так, как будто это он его написал.

Для лучшего понимания рассмотрим простой пример. Создадим консольное приложение, в котором зададим константу HelloText, и попытаемся вызвать несуществующий метод из несуществующего статического класса `HelloWorldGenerated.HelloWorld.SayHello()`. Разумеется мы не сможем сбилдить это приложение.
```C#
namespace MyApp
{
    class Program
    {
        private const string HelloText = "Hello world!";

        static void Main()
        {
            HelloWorldGenerated.HelloWorld.SayHello();
        }
    }
}
```

Теперь добавим библиотеку классов, с единственным классом `HelloWorldGenerator` в котором реализуем логику сурс генератора. В данном примере, я нахожу константу `HelloText`, беру её значение двумя разными способами, опираясь только на синтаксис, и используя данные семантической модели. Затем я генерирую класс, который хочу добавить в итоговую сборку, причем для этого не нужно использовать какие-то хитрые апи вроде `SyntaxFactory`, вручную выстраивая всё синтаксическое дерево, я просто пишу текст. И в конце я добавляю сгенерированный класс в сборку. 
```C#
using System;
using System.Collections.Generic;
using System.Text;
using System.Linq;
using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.Text; 
using Microsoft.CodeAnalysis.CSharp.Syntax;

namespace SourceGeneratorSamples
{
    [Generator]
    public class HelloWorldGenerator : ISourceGenerator
    {
        // этот метод вызывается когда происходит компиляция, из GeneratorExecutionContext
        // мы можем получить все синтаксические деревья и семантическую модель
        public void Execute(GeneratorExecutionContext context)
        {
            //находим константу HelloText из синтаксиса
            var helloTextFromSyntax = context.Compilation
                .SyntaxTrees
                .SelectMany(syntaxTree => syntaxTree.GetRoot().DescendantNodes())
                .OfType<VariableDeclaratorSyntax>()
                .Where(x => x.Identifier.ToString() == "HelloText")
                .Select(x => x.Initializer.Value)
                .OfType<LiteralExpressionSyntax>()
                .Single()
                .Token
                .ValueText;

            //находим ту же константу HelloText из семантической модели
            var helloTextFromSemantic = context.Compilation
                .GetSymbolsWithName("HelloText")
                .OfType<IFieldSymbol>()
                .Single()
                .ConstantValue;
                
            //генерирую код класса HelloWorld который реализует метод SayHello()
            var generatedSource = @"
using System;
namespace HelloWorldGenerated
{
    public static class HelloWorld
    {
        public static void SayHello() 
        {
            Console.WriteLine(""" + helloTextFromSyntax + @""");
            Console.WriteLine(""" + helloTextFromSemantic + @""");
        }
    }
}
";
            //добавлем его в сборку
            context.AddSource("helloWorldGenerated", generatedSource);
        }

        public void Initialize(GeneratorInitializationContext context)
        {
            // No initialization required
        }
    }
}

```

Добавляю к консольному приложению ссылку на проект с генератором
```xml
<ItemGroup>
    <ProjectReference Include="path-to-sourcegenerator-project.csproj" OutputItemType="Analyzer" ReferenceOutputAssembly="false" />
</ItemGroup>
```

Теперь, когда мы соберем наше консольное приложение, сгенерируется вот такой вот класс и добавиться в нашу сборку.
```C#
using System;

namespace HelloWorldGenerated
{
    public static class HelloWorld
    {
        public static void SayHello()
        {
            Console.WriteLine("Hello world!");
            Console.WriteLine("Hello world!");
        }
    }
}
```

Метод HelloWorldGenerated.HelloWorld.SayHello() станет доступным, и запустив приложение мы увидим:
```
Hello world!
Hello world!
```

Если мы изменим константу `HelloText`, класс `HelloWorld` перегенерируется.
```C#
namespace MyApp
{
    class Program
    {
        private const string HelloText = "Hello world from source generator!";

        static void Main()
        {
            HelloWorldGenerated.HelloWorld.SayHello();
        }
    }
}
```
Важно понимать, что сурс генератор работает не только когда мы жмем на кнопку билда, компиляция происходит постоянно, пока мы пишем код.

Теперь, когда у вас есть представление о том, что такое сурс генераторы, я могу рассказать о том, как работает моя реализация маппера.
Для начала генерируется класс с методами-заглушками, он добавляется сразу при подключении нугет пакета, ещё до компиляции, поэтому пользователю сразу становится доступен метод `Map`.
```C#
using System;

namespace NextGenMapper
{
    internal static partial class Mapper
    {
        internal static To Map<To>(this object source) => throw new InvalidOperationException($""Error when mapping {source.GetType()} to {typeof(To)}, mapping function was not found. Create custom mapping function."");
    }
}
```

Так как этот метод является методом расширения с параметром типа object, при подключении пространства имён `NextGenMapper` мы можем вызвать его для любого интересующего нас объекта.
```C#
using NextGenMapper;

var source = new UserSource("Vasya", "Pupkin", new DateTime(2007, 01, 01));

// например для переменной source типа UserSource
var destination = source.Map<UserDestination>();

Console.WriteLine(destination);

public record UserSource(string FirstName, string SecondName, DateTime Birthday);
public record UserDestination(string FirstName, string SecondName, DateTime Birthday);
```

Теперь в дело вступает сам генератор, он видит, что метод `Map` вызван, в качестве аргумента передана переменная `source` типа `UserSource`, в качестве аргумента типа передан тип `UserDestination`, и на основе данных об этих типах он генерирует новый метод, который помещает в тот же частичный класс `Mapper`.
```C#
using NextGenMapper.Extensions;

namespace NextGenMapper
{
    internal static partial class Mapper
    {
        internal static UserDestination Map<To>(this UserSource source) 
            => new UserDestination(source.FirstName, source.SecondName, source.Birthday);
    }
}
```

Теперь получается что в классе Mapper у нас есть два метода-кандидата: `To Map<To>(this object source)` и `UserDestination Map<To>(this UserSource source)`. Между двумя этими методами компилятор выберет второй, т.к. его аргументы более специфичны. Механика выбора лучшей функции описана в спецификации языка и на неё можно положиться.
В итоге, при запуске приложение будет вызван сгененрированный метод.

С методом `MapWith` всё чуть-чуть сложнее. Изначально у нас есть вот такой метод-заглушка, он не отличается от заглушки для метода `Map`.
```C#
using System;

namespace NextGenMapper
{
    internal static partial class Mapper
    {
        internal static To MapWith<To>(this object source) => throw new InvalidOperationException($""Error when mapping {source.GetType()} to {typeof(To)}, mapping function was not found. Create custom mapping function."");
    }
}
```

Когда вызываем его в коде, сурс генератор создает ещё один метод-заглушку
```C#
using NextGenMapper;
using System;

var source = new UserSource("Vasya", "Pupkin", "vasya@mail.ru", 19);

var destination = source.MapWith<UserDestination>();

Console.WriteLine(destination);

public record UserSource(string FirstName, string LastName, string Login, int Age);
public record UserDestination(string Name, string Email, int Age);
```

В этом методе добавлены параметры соответствующие свойствам типа `UserDestination`. Это нужно для того, чтобы IDE подсказывало нам, какие аргументы я могу передать в метод `MapWith`
```C#
using NextGenMapper.Extensions;

namespace NextGenMapper
{
    internal static partial class Mapper
    {
        internal static UserDestination MapWith<To>(
            this UserSource source,
            string name = default,
            string email = default,
            int age = default)
        {
            throw new System.NotImplementedException("This method is a stub and is not intended to be called");
        }
    }
}
```

Теперь мы можем указать аргументы для метода `MapWith`
```C#
using NextGenMapper;
using System;

var source = new UserSource("Vasya", "Pupkin", "vasya@mail.ru", 19);

var destination = source.MapWith<UserDestination>(
    name: source.FirstName + " " + source.LastName,
    email: source.Login);

Console.WriteLine(destination);

public record UserSource(string FirstName, string LastName, string Login, int Age);
public record UserDestination(string Name, string Email, int Age);
```

И сурс генератор добавит к мапперу ещё один метод, на этот раз не заглушку, а реальный метод для маппинга.
```C#
using NextGenMapper.Extensions;

namespace NextGenMapper
{
    internal static partial class Mapper
    {
        internal static UserDestination MapWith<To>(this UserSource source, string name, string email)
            => new UserDestination(name, email, source.Age);
    }
}
```
При запуске приложения компилятор, как и в случае с методом `Map`, вызовет нужный нам метод, т.к. он является лучшим кандидатом.

Как я уже говорил, сурс генератор запускается каждый раз при компиляции, и для этого не нужно нажимать кнопку билда. Может показаться что этапов генерации метода `MapWith` много, но на самом деле это происходит достаточно быстро, чтобы вы не обращали на это внимание.

## Главные отличия
Теперь, когда вы представляете как работают мапперы, можно подробнее поговорить о том, чем они отличаются.

### Статический анализ
Первое, что бросается в глаза - это поддержка статического анализа. Автомаппер создает свои методы в рантайме, и статический анализатор не может заранее проанализировать методы для маппинга. Попробуйте для какого-нибудь свойста в типе, в который вы маппите, посмотреть все ссылки на него (Find All References). С автомаппером вы ничего не увидите, потому что в этот момент не существует методов, которые используют это свойство. 

Моя реализация маппера добавляет методы в код пользователя, и мне ничего не мешает посмотреть ссылки на то или иное свойство, и перейти к сгенерированным методам которые их используют. К тому же это делает невозможной ситуацию, когда с одной стороны у нас автомаппер, с другой ORM или какой-нибудь сериализатор, и статический анализатор сообщает что у нас в коде ошибка, потому что свойство нигде не используется. 

### Возможность отладки
Один из самых неприятных моментов в использовании автомаппера - это рантайм исключения. Ты не можешь даже примерно понять, будет ли работать маппинг, пока не запустишь приложение. На первый взгляд пример ниже выглядит вполне валидным.
```C#
using AutoMapper;
using System;

var config = new MapperConfiguration(cfg =>
{
    cfg.CreateMap<UserSource, UserDestination>()
        .ForMember(dst => dst.Name, opt => opt.MapFrom(src => src.FirstName + ' ' + src.SecondName));
});
var mapper = config.CreateMapper();

var source = new UserSource("Vasya", "Pupkin", "vasya@mail.ru");

var destination = mapper.Map<UserSource, UserDestination>(source);

Console.WriteLine(destination);

public record UserSource(string FirstName, string SecondName, string Email);
public record UserDestination(string Name, string Email);
```
Но если мы запустим приложение и вызовем метод маппинга - выбросится исключение: `System.ArgumentException: 'UserDestination needs to have a constructor with 0 args or only optional args. Validate your configuration for details.'`. Для борьбы с этой проблемой нам предлагают использовать метод `AssertConfigurationIsValid()` для валидации конфигурации, проблема лишь в том, что его нужно постоянно вызывать. Можно добавить его в юнит тесты и вызывать на CI (мы так и делали, это действительно немного спасает), но это всё равно далеко от идеала. 

Иногда ошибки не возникают там, где мы могли бы этого ожидать
```C#
using AutoMapper;
using System;

var config = new MapperConfiguration(cfg =>
{
    cfg.CreateMap<UserSource, UserDestination>()
        .ForMember(dst => dst.City, opt => opt.MapFrom(src => src.Address.City));
});
var mapper = config.CreateMapper();

var source = new UserSource("Vasya", null);

var destination = mapper.Map<UserSource, UserDestination>(source);

Console.WriteLine(destination);

public record UserSource(string Name, UserAddress Address);
public record UserAddress(string City);

public record UserDestination(string Name)
{
    public string City { get; set; }
};
```
В данном примере свойство `Address` переменной `source` равно `null`. Мы указали что для маппинга в `UserDestination` нам нужно обратится к этому свойству и получить из него `City`. При этом мы ожидаем увидеть ошибку NullReferenceException, но когда мы запустим код...
```
UserDestination { Name = Vasya, City =  }
```
Ошибки нет, свойство City осталось пустым. И мы не можем обезопасить себя от такого поведения, сделав `City` только для чтения, с инициализацией через конструктор, автомаппер просто не позволит нам этого сделать. Автомаппер проглотил исключение и просто проигнорировал свойство - и для него это вполне нормальное поведение.

Ещё один пример неожиданного поведения я обнаружил случайно, пока писал эту статью. Вот вам загадка от Жака Фреско, что произойдет если выполнить этот код?
```C#
using AutoMapper;
using System;

var config = new MapperConfiguration(cfg =>
{
    cfg.CreateMap<UserSource, UserDestination>()
        .ForMember(dst => dst.Name, opt => opt.MapFrom(src => src.Name.ToLower()));
});
var mapper = config.CreateMapper();

var source = new UserSource("VASYA");

var destination = mapper.Map<UserSource, UserDestination>(source);

Console.WriteLine(destination);

public record UserSource(string Name);
public record UserDestination(string Name);
```
Правильный ответ - свойство `Name` смаппится, но функция `ToLower()` не будет применена.
```
UserDestination { Name = VASYA }
```

Возможно это баг, который исправят в будущем, но я оставил этот пример для того, чтобы вам было видно яснее - код, который написан для конфигурации маппинга не вызывается напрямую. Автомаппер в методе `MapFrom`, в качестве параметра принимает не `Func<UserSource, TSourceMember>` а `Expression<Func<TSource, TSourceMember>>`. Мы не можем ожидать, что будет выполнена логика, которую мы там написали, и проверить мы ничего не сможем. Точнее сказать, не сможем проверить быстро и легко, но стоит отметить, что способ всё-таки есть. Мы можем вызвать у конфигурации метод `BuildExecutionPlan()`, который вернет нам дерево выражений, построенное автомаппером. А чтобы читать его было проще, нам предлагают использовать расширение для IDE. Наверно, никто не будет спорить с тем, что нажать F12 (Go To Definition) на интересующем нас методе `Map` и посмотреть код который будет вызван, намного проще и быстрее, да и мы привыкли делать именно так.

Если посмотреть что будет выполняться при маппинге мы ещё как-то можем, то узнать как - нет. Мы не можем поставить точку останова на методе `Map` и войти в него. Нужно ли объяснять насколько это неудобно. Если в вашем коде для маппинга где-то ошибка, вы не узнаете об этом вызвав метод для валидации конфигурации, вы не узнаете об этом и от самого автомаппера, потому что он проглотит эту ошибку, единственный шанс - заметить что свойство не смаппилось. И когда вы это заметите - молитесь, чтобы вы сразу обнаружили ошибку, при взгляде на дерево выражений, потому что иного способа её обнаружить не представится.

Как же должен вести себя маппер в подобных ситуациях? Хотелось бы, чтобы маппер сразу, во время компиляции сообщал нам о всех проблемах. Он должен следить за тем, что он сконфигурирован валидно и используется правильно, в ином случае, мы должны получить ошибку во время компиляции проекта.
```C#
using NextGenMapper;
using System;

var source = new UserSource("Vasya", "Pupkin", "vasya@mail.ru");

var destination = source.MapWith<UserDestination>();

Console.WriteLine(destination);

public record UserSource(string FirstName, string SecondName, string Email);
public record UserDestination(string Name, string Email);
```
Например в данном случае мы не сможем запустить приложение, потому что билд упадет с ошибкой: `NGM005: Method 'MapWith' must be called at least with one argument. Pass arguments to 'MapWith' method or use 'Map' method.`.

Ещё несколько примеров
```C#
using NextGenMapper;
using System;

var source = new UserSource("Vasya", new DateTime(2010, 10, 20));

var destination = source.Map<UserDestination>();

Console.WriteLine(destination);

public record UserSource(string Name, DateTime Birthdate);
public record UserDestination(string Name, string Birthdate);
```
Таже история, билд падает с ошибкой: `NGM009: Function for mapping 'UserSource.Birthdate' of type 'System.DateTime' to 'UserDestination.Birthdate' of type 'string' was not found. Add custom mapping function for mapper`

```C#
using NextGenMapper;
using System;

var source = new UserSource("Vasya");

var destination = source.Map<UserDestination>();

Console.WriteLine(destination);

public record UserSource(string Name);
public record UserDestination(string Name, string Email);
```
Ошибка: `NGM002: Constructor for mapping from UserSource to UserDestination was not found, make sure that the UserDestination has a public constructor whose parameters correspond to the properties of the UserSource.`

Разумеется, избавиться от всех рантайм исключений не получится, но стремиться к этому необходимо.
```C#
using NextGenMapper;
using System;

var source = new UserSource("Vasya", null);

var destination = source.MapWith<UserDestination>(city: source.Address.City);

Console.WriteLine(destination);

public record UserSource(string Name, UserAddress? Address);
public record UserAddress(string City);

public record UserDestination(string Name, string City);
```
В этом примере мой маппер волшебства не покажет, и при вызове метода, выбросит исключение `System.NullReferenceException: 'Object reference not set to an instance of an object.`

В большинстве случаев будет достаточно того, что маппер не проглатывает исключения, а код вызываемого метода можно посмотреть одним нажатием кнопки, но если вдруг понадобится по шагам пройти весь маппинг - такая возможность также имеется. Работает это не идеально, потому что идеально было бы поставить точку останова прямо в сгенерированном методе, но пока что такой возможности нет. Но можно поставить точку останова на самом вызове метода для маппинга, и при вызове метода, войти в него с помощью F11 (Step Into), тогда мы попадем внутрь метода и сможем отлаживать его как свой собственный код.

### Производительность

Я не буду долго останавливаться на производительности своего маппера, показывать результаты бенчмарков, сравнивая между собой. Понятное дело, что автомаппер проиграет, а в некоторых случаях, проиграет и маппер, написанный вручную, потому что когда мы пишем его руками, мы не очень сильно заморачиваемся над производительностью, нам важнее читаемый код и простота его написания, а когда методы генерируются автоматически, можно что-нибудь придумать.

Для примера мы хотим смаппить `List<UserSource>` в `UserDestination[]`
```C#
using NextGenMapper;
using System;
using System.Collections.Generic;

var source = new UserSource("Vasya");
var sourceList = new List<UserSource> { source };

var destinationArray = sourceList.Map<UserDestination[]>();

Console.WriteLine(destinationArray[0]);

public record UserSource(string Name);
public record UserDestination(string Name);
```

Будет сгенерирован следующий код
```C#
using NextGenMapper.Extensions;

namespace NextGenMapper
{
    internal static partial class Mapper
    {
        internal static UserDestination Map<To>(this UserSource source) => new UserDestination(source.Name);

        internal static UserDestination[] Map<To>(this System.Collections.Generic.List<UserSource> source)
        {
            var destination = new UserDestination[source.Count];
            #if NET5_0_OR_GREATER
            var sourceSpan = System.Runtime.InteropServices.CollectionsMarshal.AsSpan(source);
            #endif
            for (var i = 0; i < source.Count; i++)
            {
                #if NET5_0_OR_GREATER
                destination[i] = sourceSpan[i].Map<UserDestination>();
                #else
                destination[i] = source[i].Map<UserDestination>();
                #endif
            }

            return destination;
        }
    }
}
```
Я же обычно использовал LINQ метод `Select`. Он в разы медленнее и использует больше памяти. 

## Итог

После своего опыта с автомаппером, мне хотелось создать инструмент, который не будет требовать много времени, чтобы в нем разобраться, у которого нет огромного количества функций, нескольких разных способов подключения, настройки, и использования. Я хотел максимально простое решение. И главное, чтобы этот инструмент решал проблемы, а не создавал новые. Мне хотелось свести к минимуму время и силы, потраченные на написание кода, не касающегося моих задач, на попытки разобраться почему инструмент не работает так, как от него ожидаешь. Пока что я не достиг своей цели, но мне кажется я двигаюсь в правильном направлении.

Если вам интересно самостоятельно опробовать написанный мной маппер, переходите на [страницу проекта на github](https://github.com/DedAnton/NextGenMapper). Прошу с пониманием отнестись к возможным проблемам, это пока лишь альфа версия, а не готовый продукт. 

Буду рад услышать конструктивную критику, а также обсудить идеи и предложения в комментариях.

### Ссылки.
[C# Source Generators overview](https://docs.microsoft.com/en-us/dotnet/csharp/roslyn-sdk/source-generators-overview)
[Как определяется лучшая функция-кандидат](https://github.com/dotnet/csharplang/blob/a4c9db9a69ae0d1334ed5675e8faca3b7574c0a1/spec/expressions.md#better-function-member)