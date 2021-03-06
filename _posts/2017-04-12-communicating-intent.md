---
title: Передача намерений
author: Jasper 'jaheba' Schulz
original: https://github.com/jaheba/stuff/blob/master/communicating_intent.md
translator: Илья Богданов
categories: обучение
---

{% img '2017-04-12-communicating-intent/teaser.png' alt:'teaser' %}

Rust — элегантный язык, который несколько отличается от многих других популярных
языков. Например, вместо использования классов и наследования, Rust предлагает
собственную систему типов на основе типажей. Однако я считаю, что многим
программистам, начинающим свое знакомство с Rust (как и я), неизвестны
общепринятые шаблоны проектирования.

В этой статье, я хочу обсудить шаблон проектирования *новый тип* (newtype), а
также типажи `From` и `Into`, которые помогают в преобразовании типов.

<!--cut-->

Скажем, вы работаете в европейской компании, создающей замечательные цифровые
термостаты для обогревателей, готовые к использованию в Интернете Вещей. Чтобы
вода в обогревателях не замерзала (и не повреждала таким образом обогреватели),
мы гарантируем в нашем программном обеспечении, что если есть опасность
замерзания, мы пустим по радиатору горячую воду. Таким образом, где-то в нашей
программе есть следующая функция:

```rust
fn danger_of_freezing(temp: f64) -> bool;
```

Она принимает некоторую температуру (полученную с датчиков по Wi-Fi) и управляет
потоком воды соответствующим образом.

Все идет отлично, покупатели довольны и ни один обогреватель в итоге не
пострадал. Руководство решает перейти на рынок США, и вскоре наша компания
находит местного партнера, который связывает свои датчики с нашим замечательным
термостатом.

Это катастрофа.

После расследования выясняется, что американские датчики передают температуру в
градусах Фаренгейта, в то время как наше программное обеспечение работает с
градусами Цельсия. Программа начинает подогрев как только температура опускается
ниже 3° Цельсия. Увы, 3° по Фаренгейту ниже точки замерзания. Впрочем,
после обновления программы нам удается справиться с проблемой и ущерб составляет
всего несколько десятков тысяч долларов.
[Другим повезло меньше](https://en.wikipedia.org/wiki/Mars_Climate_Orbiter).

## Новые типы

Проблема возникла из-за того, что мы использовали числа с плавающей запятой,
имея в виду нечто большее. Мы присвоили этим числам смысл без
явного указания на это. Другими словами, наше намерение заключалось в работе
именно с единицами измерения, а не с обычными числами.
Типы, на помощь!

```rust
#[derive(Debug, Clone, Copy)]
struct Celsius(f64);

#[derive(Debug, Clone, Copy)]
struct Fahrenheit(f64);
```

Программисты, пишущие на Rust, называют это шаблоном проектирования `новый тип`.
Это структура-кортеж, содержащая единственное значение. В этом примере мы
создали два новых типа, по одному для градусов Цельсия и Фаренгейта.

Наша функция приобрела такой вид:

```rust
fn danger_of_freezing(temp: Celsius) -> bool;
```

Использование её с чем-либо кроме градусов Цельсия приводит к ошибкам
во время компиляции. Успех!

### Преобразования

Все что нам остается - это написать функции преобразования, которые будут
переводить одни единицы измерения в другие.

```rust
impl Celsius {
    to_fahrenheit(&self) -> Fahrenheit {
        Fahrenheit(self.0 * 9./5. + 32.)
    }
}

impl Fahrenheit {
    to_celsius(&self) -> Celsius {
        Celsius((self.0 - 32.) * 5./9.)
    }
}
```

А потом использовать их, например, так:

```rust
let temp: Fahrenheit = sensor.read_temperature();
let is_freezing = danger_of_freezing(temp.to_celsius());
```

## From и Into

Преобразования между различными типами - обычное дело в Rust. Например, мы можем
превратить `&str` в `String`, используя `to_string`, например:

```rust
// "Привет" имеет тип &'static str
let s = "Привет".to_string();
```

Однако, также возможно использовать `String::from` для создания строк так:

```rust
let s = String::from("привет");
```

Или даже так:

```rust
let s: String = "привет".into();
```

Зачем же все эти функции, когда они, на первый взгляд, делают одно и то же?

### В дикой природе
*Примечание переводчика: в этом заголовке содержалась непереводимая игра слов.
Оригинальное название **Into the Wild** можно перевести как "В дикой природе", а
можно "Великолепный `Into`"*

Rust предлагает типажи, которые унифицируют преобразования из одного типа в
другой. `std::convert` описывает, помимо других, типажи `From` и `Into`.

```rust
pub trait From<T> {
    fn from(T) -> Self;
}

pub trait Into<T> {
    fn into(self) -> T;
}
```

Как можно увидеть выше, `String` реализует `From<&str>`, а `&str` реализует
`Into<String>`. Фактически, достаточно реализовать один из этих типажей, чтобы
получить оба, так как можно считать, что это одно и то же. Точнее,
[From реализует Into](https://doc.rust-lang.org/src/core/convert.rs.html#275).

Так что давайте сделаем то же самое для температур:

```rust
impl From<Celsius> for Fahrenheit {
    fn from(c: Celsius) -> Self {
        Fahrenheit(c.0 * 9./5. + 32.)
    }
}

impl From<Fahrenheit> for Celsius {
    fn from(f: Fahrenheit) -> Self {
        Celsius((f.0 - 32.) * 5./9. )
    }
}
```

Применяем это в нашем вызове функции:

```rust
let temp: Fahrenheit = sensor.read_temperature();
let is_freezing = danger_of_freezing(temp.into());
// или
let is_freezing = danger_of_freezing(Celsius::from(temp));

```

### Слушаюсь и повинуюсь
*Примечание переводчика: оригинальное название **Your wish is my command** -
устойчивое выражение, аналог русского "Слушаюсь и повинуюсь"*

Вы можете возразить, что мы получили не так уж много преимуществ от
типажа `From`, по сравнению реализацией функций преобразования вручную, как
делали раньше. Можно даже утверждать обратное, что `into` - гораздо менее
очевидно, чем `to_celsius`.

Давайте переместим преобразование величин внутрь функции:

```rust
// T - любой тип, который можно перевести в градусы Цельсия
fn danger_of_freezing<T>(temp: T) -> bool
where T: Into<Celsius> {
    let celsius = Celsius::from(temp);
    ...
}
```

Эта функция волшебным образом принимает и градусы Цельсия, и Фаренгейта,
оставаясь при этом типобезопасной:

```rust
danger_of_freezing(Celsius(20.0));
danger_of_freezing(Fahrenheit(68.0));
```

Мы можем пойти еще дальше. Можно не только обрабатывать множество преобразуемых
типов, но и возвращать значения разных типов схожим образом.

Допустим, нам нужна функция, которая возвращает точку замерзания. Она должна
возвращать градусы Цельсия или Фаренгейта - в зависимости от контекста.

```rust
fn freezing_point<T>() -> T
where T: From<Celsius> {
    Celsius(0.0).into()
}
```

Вызов этой функции немного отличается от других функций, где мы точно знаем
возвращаемый тип. Здесь же мы должны *запросить* тип, который нам нужен.

```rust
// вежливо просим градусы Фаренгейта
let temp: Fahrenheit = freezing_point();
```

Есть второй, более явный способ вызвать функцию:

```rust
// вызываем функцию, которая возвращает градусы Цельсия
let temp = freezing_point::<Celsius>();
```

### Упакованные (boxed) значения

Эта техника не только полезна для преобразования величин друг в друга, но также
упрощает обработку упакованных значений, например результатов из
[баз данных](https://github.com/sfackler/rust-postgres)

```rust
let name: String = row.get(0);
let age: i32 = row.get(1);

// вместо
let name = row.get_string(0);
let age = row.get_integer(1);
```

## Заключение

У Python есть замечательный [Дзен](https://www.python.org/dev/peps/pep-0020/).
Его первые две строки гласят:
> Красивое лучше, чем уродливое.
> Явное лучше, чем неявное.

Программирование - это акт передачи намерений компьютеру. И мы должны явно
указывать, что именно имеем в виду, когда пишем программы. Например,
совершенно невыразительное булево значение для указания порядка сортировки не
отразит наше намерение в полной мере. В Rust мы можем просто использовать
перечисление, чтобы избавиться от любой двусмысленности:

```rust
enum SortOrder {
    Ascending,
    Descending
}
```

Таким же образом *новые типы* помогают придать смысл простым значениям.
`Celsius(f64)` отличается от `Miles(f64)`, хотя они могут иметь одно и то же
внутреннее представление (`f64`). С другой стороны, использование `From` и `Into`
помогает нам упрощать программы и интерфейсы.
