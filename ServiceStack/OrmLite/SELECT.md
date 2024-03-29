# SELECT

## Querying with SELECT

---

```csharp
int agesAgo = DateTime.Today.AddYears(-20).Year;
db.Select<Author>(x => x.Birthday >= new DateTime(agesAgo, 1, 1) 
                    && x.Birthday <= new DateTime(agesAgo, 12, 31));

db.Select<Author>(x => Sql.In(x.City, "London", "Madrid", "Berlin"));

db.Select<Author>(x => x.Earnings <= 50);

db.Select<Author>(x => x.Name.StartsWith("A"));

db.Select<Author>(x => x.Name.EndsWith("garzon"));

db.Select<Author>(x => x.Name.Contains("Benedict"));

db.Select<Author>(x => x.Rate == 10 && x.City == "Mexico");

db.Select<Author>(x => x.Rate.ToString() == "10"); //impicit string casting

db.Select<Author>(x => "Rate " + x.Rate == "Rate 10"); //server string concatenation
```

## Convenient data access patterns

---

```csharp
// 根据主键查询一条记录
Person person = db.SingleById<Person>(1);

Person person = db.Single<Person>(x => x.Age == 42);
```

## Using Typed SqlExpression

---

The `From<T>` can be used to create an `SqlExpression<T>` query to build on and use later.
```csharp
var q = db.From<Person>()
          .Where(x => x.Age > 40)
          .Select(Sql.Count("*"));

int peopleOver40 = db.Scalar<int>(q);
```

    Scalar 返回单行单列

Common aggregate methods can be used with type safety. For example:
```csharp
int peopleUnder50 = db.Count<Person>(x => x.Age < 50);

bool has42YearOlds = db.Exists<Person>(new { Age = 42 });

// 查询Person返回int
int maxAgeUnder50 = db.Scalar<Person, int>(x => Sql.Max(x.Age), x => x.Age < 50);
```

Returning a single column from a query can be used with `.Select` of a property and `.Column`.
    Column 返回首列
```csharp
var q = db.From<Person>()
    .Where(x => x.Age == 27)
    .Select(x => x.LastName);
    
List<string> results = db.Column<string>(q);

var q = db.From<Person>()
          .Where(x => x.Age < 50)
          .Select(x => x.Age);

HashSet<int> results = db.ColumnDistinct<int>(q);
```

Multiple columns can do the same. For example using the same `.Select` on a `SqlExpression<T>` with an anonymous type, returning a Dictionary of matching types.
```csharp
var q = db.From<Person>()
          .Where(x => x.Age < 50)
          .Select(x => new { x.Id, x.LastName });

Dictionary<int,string> results = db.Dictionary<int, string>(q);
```

The `Lookup<T,K>` method returns a `Dictionary<K,List<V>>` grouping made from the first two columns using n SQL Expression.
```csharp
var q = db.From<Person>()
          .Where(x => x.Age < 50)
          .Select(x => new { x.Age, x.LastName });
// age为key，value为根据key分组后的数组
Dictionary<int, List<string>> results = db.Lookup<int, string>(q);
```

The `db.KeyValuePair<K,V>` API is similar to `db.Dictionary<K,V>` where it uses the **first 2 columns** for its Key/Value Pairs to create a Dictionary but is more appropriate when the results can contain duplicate Keys or when ordering needs to be preserved:

    KeyValuePair<K,V> 和 Dictionary<K,V>很相似，当value为分组后的结果或者，需要保留排序顺序时使用KeyValuePair<K,V>更合适

```csharp
var q = db.From<StatsLog>()
    .GroupBy(x => x.Name)
    .Select(x => new { x.Name, Count = Sql.Count("*") })
    .OrderByDescending("Count");

var results = db.KeyValuePairs<string, int>(q);
```

## Lambda Expression examples

---

For simple queries you can use terse lambda Expressions to specify the filter conditions you want:
```csharp
var nirvana = db.Select<Artist>(x => x.Name == "Nirvana").First();

var nirvanaTracks = db.Select<Track>(x => x.ArtistId == nirvana.Id);

// nirvanaTracks 中截取为idList
var nirvanaTrackIds = nirvanaTracks.Map(x => x.Id);

//Using SQL IN by .NET Collection `Contains()` or explicit `Sql.In()`
var nirvanaTracksByIn = db.Select<Track>(x => nirvanaTrackIds.Contains(x.Id));
var nirvanaTracksByInAlt = db.Select<Track>(x => Sql.In(x.Id, nirvanaTrackIds));

var pearlJam = db.Select<Artist>(x => x.Name.StartsWith("Pearl")).First();

var faithNoMore = db.Select<Artist>(x => x.Name.EndsWith("More")).First();

var smellsLikeTeenSpirit = db.Select<Track>(x => x.Name.Contains("Teen")).First();

var latestTracks = db.Select<Track>(x => x.Year >= 1997);

var heartShapedBox = db.Select<Track>(x => x.ArtistId == nirvana.Id 
	&& x.Year == 1993 && x.Album == "In Utero").First();
```

## SqlExpression examples

---

```csharp
var q = db.From<Track>()
    .OrderByDescending(x => x.Year)
    .Take(3);
var latest3Tracks = db.Select(q);

var faithAndLiveTracks = db.Select(db.From<Track>()
    .Where(x => x.Album == "Angel Dust" && x.Year == 1992)
    .Or(x => x.Album == "Throwing Copper" && x.Year == 1994));

// More advanced SQL Expression
var customYears = new[] { 1993, 1994, 1997 };
q = db.From<Track>()
    .Where(x => customYears.Contains(x.Year))
    .And(x => x.Name.Contains("A"))
    .GroupBy(x => x.Year)
    .OrderByDescending("Total")
    .ThenBy(x => x.Year)
    .Take(2)
    .Select(x => new { x.Year, Total = Sql.Count("*") });

var top2CountOfAByYear = db.Dictionary<string, int>(q);
```

## SqlExpression with JOIN examples

---

```csharp
var q = db.From<Track>()
    .Join<Artist>() //Uses implicit reference convention
    .Where<Artist>(x => x.Name == "Nirvana");
var implicitJoin = db.Select(q);

var explicitJoin = db.Select(db.From<Track>()
	.Join<Artist>((track,artist) => track.ArtistId == artist.Id)
    .Where<Artist>(x => x.Name == "Nirvana"));

var nirvanaWithRefs = db.LoadSingleById<Artist>(explicitJoin[0].ArtistId);

var oldestTracks = db.Select(db.From<Track>()
    .Where(x => Sql.In(x.Year, db.From<Track>().Select(y => Sql.Min(y.Year)))));

var oldestTrackIds = oldestTracks.Map(x => x.Id);
var earliestArtistsWithRefs = db.LoadSelect(db.From<Artist>()
    .Where(a => oldestTracks.Map(t => t.ArtistId).Contains(a.Id)));

var oldestTracksAndArtistNames = db.Dictionary<string, string>(db.From<Track>()
	.Join<Artist>()
	.Where(x => oldestTrackIds.Contains(x.Id))
    .Select<Track,Artist>((t,a) => new { t.Name, Artist = a.Name }));

// 转为元组 oldestTrackAndArtists.Where(x=>x.item1.xxx x.item2.xxx )
var oldestTrackAndArtists = db.SelectMulti<Track,Artist>(db.From<Track>()
      .Join<Artist>()
      .Where(x => oldestTrackIds.Contains(x.Id)));
```

## Single, Scalar, Count, Exists examples

---

```csharp
var nevermind = db.Single<Track>(x => x.Album == "Nevermind");

var nirvana = db.SingleById<Artist>(nevermind.ArtistId);

var latestYear = db.Scalar<int>(db.From<Track>()
    .Select(x => Sql.Max(x.Year)));

var differentArtistsCount = db.Scalar<int>(db.From<Track>()
    .Select(x => Sql.CountDistinct(x.ArtistId)));

int tracksAfter93 = db.Scalar<Track,int>(x=> Sql.Count("*"), x=> x.Year > 1993);

var nirvanaTracksCount = db.Count<Track>(x => x.ArtistId == nirvana.Id);

$"\nHave Tracks in 1990: {db.Exists<Track>(x => x.Year == 1990)}".Print();
$"\nHave Tracks in 1991: {db.Exists<Track>(x => x.Year == 1991)}".Print();

var inUtero = db.Where<Track>(new { ArtistId = nirvana.Id, Year = 1993 });

// 延迟加载
var lazySequence = db.SelectLazy(db.From<Track>().OrderBy(x => x.Year));
var lazyLinq = lazySequence.Take(3).Select(x => $"{x.Year}: {x.Album}");
db.Insert(new Track {
    Name = "About a Girl", ArtistId = nirvana.Id, Album="Bleach", Year=1989 }); 
lazyLinq.Each(x => x.Print());
```

## Column, ColumnDistinct, Dictionary and Lookup Examples

---

```csharp
List<int> trackIds = db.Column<int>(db.From<Track>());

HashSet<int> years = db.ColumnDistinct<int>(db.From<Track>().Select(x => x.Year));

Dictionary<string, int> trackAndYears = db.Dictionary<string, int>(
    db.From<Track>().Select(x => new { x.Name, x.Year }));

var tracksCountByYear = db.Dictionary<int, int>(db.From<Track>()
	.Join<Artist>()
    .GroupBy(x => x.Year)     
    .OrderBy(x => x.Year)
    .Select(x => new { x.Year, Count = Sql.Count("*") }));

Dictionary<int, List<string>> tracksByYear = db.Lookup<int, string>(
	db.From<Track>().Select(x => new { x.Year, x.Name }));
```