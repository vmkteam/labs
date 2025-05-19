+++
title = 'Colgen'
+++

# Colgen

`colgen` - утилита для генерации различного кода и не только:
1. Стабильная (повторяемая) генерация кода
2. Генерация сниппетов (в инлайн режиме)
3. Работа с LLM (анализ кода)


## Генерация кода

Рассмотрим более подробно на примере проекта `newsportal`. Сколько раз в проектах мы встречали код вот такого типа?
```go
ids := make([]int, len(list))
for i := range list {
	ids = append(ids, i.ID)
}
```
То есть у нас есть какой-то `list`, например `[]News`, и нам нужно собрать все `ID` в слайс для дальнейшего использования.
Такие простые куски кода разрастаются по проекту, но не несут никакой смысловой нагрузки. Или рассмотрим другой пример – преобразование слайса в мап по ID:
```go
r := make(map[int]News, len(list))
for i, n := range list {
	r[n.ID] = n
}
```
И такого простого кода обычно много. Его можно упростить. Например, мы можем ввести понятие **коллекций**.

### Базовые генераторы
Создадим файл `collection.go`, в нем объявим новый тип и пару методов у него.
```go
type NewsList []News

func (ll NewsList) IDs() []int {
    r := make([]int, len(ll))
    for i := range ll {
        r[i] = ll[i].ID
    }
    return r
}

func (ll NewsList) Index() map[int]News {
    r := make(map[int]News, len(ll))
    for i := range ll {
        r[ll[i].ID] = ll[i]
    }
    return r
}
```
Теперь мы можем упросить основной код и использовать новые методы:
```go
nn := NewsList(list)
ids := nn.IDs()
idx := nn.Index()
```
**Плюсы:**
1. Утилитарный код ушел из бизнес-логики и был стандартизирован.
2. Появился новый тип, для которого мы можем добавить больше методов.
3. _Неочевидный плюс:_ стандартизируем нейминг, не нужно больше придумывать имена переменным в утилитарном коде.

**Минусы:**
1. Код все еще нужно писать.

Давайте попробуем убрать минусы и добавим в наш файл:
```go
//go:generate colgen
//colgen:News
```
Данная конструкция создаст новый файл с суффиксом `_colgen.go`, в котором будет базовый тип для структуры и два метода: `IDs()` и `Index()` (если есть поле ID).

Утилита находится тут https://github.com/vmkteam/colgen/

### Дополнительные генераторы

Если поле `ID` отсутствует (например, вместо него `NewsID`), то можно добавить следующую конструкцию для аналогичного поведения:
```go
//go:generate colgen
//colgen:News
//colgen:News:NewsID,Index(NewsID)
```
В результате мы получим базовый тип и два метода: `NewsIDs()` и `IndexByNewsID()`.
Допустим, у новости есть теги `TagIDs`, и нам из списка надо получить уникальные теги. А у тегов есть поле `alias`, и нам нужно сделать индекс по нему.
```go
type News struct {
	ID     int
	Text   string
	TagIDs []int
}

type Tag struct {
	ID    int
	Alias string
}

//go:generate colgen
//colgen:News,Tag
//colgen:News:UniqueTagIDs
//colgen:Tag:Index(Alias)
```
На выходе у нас будет вот такой файл с кодом:
```go
type NewsList []News

func (ll NewsList) IDs() []int {
	r := make([]int, len(ll))
	for i := range ll {
		r[i] = ll[i].ID
	}
	return r
}

func (ll NewsList) Index() map[int]News {
	r := make(map[int]News, len(ll))
	for i := range ll {
		r[ll[i].ID] = ll[i]
	}
	return r
}

func (ll NewsList) UniqueTagIDs() []int {
	idx := make(map[int]struct{})
	for i := range ll {
		for _, v := range ll[i].TagIDs {
			if _, ok := idx[v]; !ok {
				idx[v] = struct{}{}
			}
		}
	}

	r, i := make([]int, len(idx)), 0
	for k := range idx {
		r[i] = k
		i++
	}
	return r
}

type Tags []Tag

func (ll Tags) IDs() []int {
	r := make([]int, len(ll))
	for i := range ll {
		r[i] = ll[i].ID
	}
	return r
}

func (ll Tags) Index() map[int]Tag {
	r := make(map[int]Tag, len(ll))
	for i := range ll {
		r[ll[i].ID] = ll[i]
	}
	return r
}

func (ll Tags) IndexByAlias() map[string]Tag {
	r := make(map[string]Tag, len(ll))
	for i := range ll {
		r[ll[i].Alias] = ll[i]
	}
	return r
}
```
Теперь мы можем сконцентрироваться на решении практических задач, а утилитарный код отдать на откуп генератору.
Единственное, на что надо обращать внимание: код перед вызовом генератора **должен компилироваться**, потому что идет парсинг go файлов через AST.

#### Group(Field)
Можно сгруппировать слайс по определенному полю. На выходе получим такой код `//colgen:News:Group(CategoryID)`:
```go
func (ll NewsList) GroupByCategoryID() map[int]NewsList {
    r := make(map[int]NewsList, len(ll))
    for i := range ll {
        r[ll[i].CategoryID] = append(r[ll[i].CategoryID], ll[i])
    }
    return r
}
```

_TLDR:_ Можно собрать все или уникальные значения по любому полю структуры или построить индекс в виде мапы.

### Map & MapP
Рассмотрим следующий пример:
```go
func NewNews(n db.News) News {
	return News{
		ID:     n.ID,
		Text:   n.Text,
		TagIDs: n.TagIDs,
	}
}

func NewNewsList(nn []db.News) []News {
	r := make([]News, len(nn))
	for i := range nn {
		r[i] = NewNews(nn[i])
	}
	return r
}
```
* `NewNews` – конструктор, который преобразует структуру из одного слоя в другой.
* `NewNewsList` – конструктор для слайса.

В силу многослойной архитектуры подобного кода будет встречаться в проекте много.
Давайте попробуем решить проблему с `NewNewsList`. Расширим наш последний вызов //go:generate
```
//go:generate colgen -imports=newsportal/pkg/db
//colgen:News,Tag
//colgen:News:UniqueTagIDs,Map(db)
//colgen:Tag:Index(Alias)
```
Здесь мы добавили `Map(db)`. В результате получаем новую функцию в сгенерированном файле:  
```go
func NewNewsList(in []db.News) NewsList { return Map(in, NewNews) }
```
* Если `NewNews` принимает `*db.News`, то необходимо использовать `MapP(db)`.
* Если нужны приватные конструкторы, то используем `mapp/map` вместо `MapP/Map`
* Если название структуры в `db` слое отличается, то используем полный путь: `Map(db.News)`
* Если структура не описана в базовой генерации `//colgen:News,Tags,...`, то конструктор будет возвращать `[]News` вместо `NewsList`
* Для стабильной генерации добавляем импорт в файл через флаг: ` -imports=newsportal/pkg/db`
* Если `Map/MapP` лежат в другом пакете, то нужно добавить этот пакет в импорт через запятую и добавить название пакета через `-funcpkg=<pkg>`.
* Не забудем добавить в проект те самые два дженерика:
```go

// MapP converts slice of type T to slice of type M with given converter with pointers.
func MapP[T, M any](a []T, f func(*T) *M) []M {
	n := make([]M, len(a))
	for i := range a {
		n[i] = *f(&a[i])
	}
	return n
}

// Map converts slice of type T to slice of type M with given converter.
func Map[T, M any](a []T, f func(T) M) []M {
	n := make([]M, len(a))
	for i := range a {
		n[i] = f(a[i])
	}
	return n
}
```
Рекомендуется иметь один `colgen` на пакет. Директивы `//go:generate colgen` можно размещать в файле `collection.go`.

>Если все сломалось, то удаляйте файл `_colgen.go` и вызывайте генератор снова. Но! Если функции генератора уже используются в коде, то при удалении файла будет нерабочий код и генератор не запустится. 

Таким образом можно описать генерацию конструкторов для слайсов. Но что делать с конструкторами?

## Генерация сниппетов в инлайн режиме

```go
//go:generate colgen
...
//colgen@NewNews(db)
```

При генерации директива `//colgen@NewNews(db)` **будет заменена в этом файле** на следующий код:
```go
type News struct { 
    db.News
}

func NewNews(in *db.News) *News {
    if in == nil {
        return nil
    }
    
    return &News{ 
        News: *in,
    }
}
```

Второй пример `//colgen@newNewsSummary(db.News,full,json)`:
```go
type NewsSummary struct {
    ID     int    `json:"newsSummaryId"`
    Text   string `json:"text"`
    TagIDs []int  `json:"tagIDs"`
}

func newNewsSummary(in *db.News) *NewsSummary {
    if in == nil {
        return nil
    }
    
    return &NewsSummary{
        ID:     in.ID,
        Text:   in.Text,
        TagIDs: in.TagIDs,
    }
}
```
Рассмотрим оба случая. 

Когда мы говорим о доменном слое, то нам подходит первый вариант.

Когда мы говорим о слое с апи, нам подходит второй вариант.

На сложных объектах вы будете получать невалидный сниппет. Но базовая идея заключается в том, чтобы получить сниппет, а потом его отредактировать как нужно. Все конструкторы уникальны, поэтому невозможно сделать их стабильную генерацию. Но генерация сниппетов может помочь!

И коротко об embed структурах в доменном слое: нет особого логического смысла дублировать поля, поэтому можно просто встроить структуру и допустить меньше ошибок в будущем при ее изменении.

Теперь попробуем описать шаги для максимальной генерации:
1. Делаем базовые типы и методы.
2. Создаем сниппеты конструктора через инлайн режим.
3. Добавляем себе в проект `Map/MapP` дженерики.
4. В последнюю очередь добавляем `Map/MapP` генераторы. Если их добавить в пункте 1, то будет некомпилируемый код из-за отсутствия функций конструктора.

## Работа с LLM

Для работы с `DeepSeek` или `Claude` у вас должен быть API ключ. \$2 или \$5 придется потратить. Локальный режим не поддерживается, PR приветствуются.
Работа с LLM идет в рамках файла, именно его содержимое (+тест, если применимо) + промт отправляется на сервер. Зависимые типы / пакеты или файлы не отправляются.

Есть три режима работы:
* `//colgen@ai:review` – код ревью с идеоматичным промтом. Результат будет рядом в файле с суффиксом `.md`. Бонусом будет предложена отсутствующая документация у методов и структур.
* `//colgen@ai:readme` – генерация README.md по текущему файлу. Результат будет рядом в файле с суффиксом `.md`.
* `//colgen@ai:tests` Генерация тестов с нуля или путем дописывания файла с тестами по образу и подобию текущих тестов в файле `_test.go`. 

Для выбора LLM в конце нужно добавить `(deepseek)` или `(claude)`. DeepSeek используется по умолчанию, поэтому его можно не добавлять.
Примеры:
```
//go:generate colgen
//colgen@ai:readme           // makes readme using deepseek by default
//colgen@ai:tests(deepseek)  // makes tests using deepseek explicitly
//colgen@ai:review(claude)   // makes review using claude
```

Примеры:
* ревью файла [colgen.go](https://github.com/vmkteam/colgen/blob/master/cmd/colgen/colgen.go)
  * [DeepSeek](review-deepseek/)
  * [Claude](review-claude/)
* [README.md](https://github.com/vmkteam/colgen/blob/master/README.md) – на 95% сгенерирован.
* [assistant_test.go](https://github.com/vmkteam/colgen/blob/master/pkg/colgen/assistant_test.go) – на 100% сгенерирован.