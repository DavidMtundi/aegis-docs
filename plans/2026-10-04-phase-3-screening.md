# Phase 3 — Sanctions and PEP Screening Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** screen every customer of a bank or SACCO against the UN Security Council consolidated list, the US OFAC SDN list and the institution's own uploaded list (PEPs, the Kenya domestic list, internal blacklists), let a reviewer confirm or dismiss each possible match, and turn confirmed matches into alerts and risk points.

**Architecture:** the empty `Aegis.Modules.Screening` project gets a pure matching core (name normalizer, Jaro-Winkler, order-insensitive name scorer, an in-memory `WatchlistIndex`) and parsers for the UN and OFAC XML files. UN and OFAC entries are global tables shared by all tenants, refreshed daily from the official URLs; tenant lists and matches are tenant-owned. `CustomerScreeningService` (in `Aegis.Application`) screens one customer or a whole tenant, records possible matches for review, and on confirmation raises an alert under a fixed "screening" rule id and rescores the customer. New risk factors make a confirmed sanctions match put the customer in the CRITICAL band.

**Tech Stack:** .NET 9, EF Core 9 + Npgsql, `System.Xml.Linq` (built in), xUnit 2.9, `WebApplicationFactory`; Next 16 / React 19 console with the existing `components/ui` kit.

## Decisions (owner, 2026-10-04)

- First customers: banks and SACCOs.
- Lists: UN consolidated list, OFAC SDN list, and one tenant-uploaded CSV list (categories `PEP`, `SANCTIONS`, `WATCHLIST`). EU/UK lists later.
- Confirmed match: raises an alert and adds risk points; no automatic blocking. Severity is CRITICAL for `SANCTIONS` and `WATCHLIST` matches and HIGH for `PEP` matches, because being a PEP calls for enhanced due diligence rather than an immediate freeze. Change `ScreeningAlertPolicy` (Task 6) if the owner wants CRITICAL for PEPs too.
- Risk: new factors `SANCTIONS_MATCH` (weight 100, so the score caps at 100 = CRITICAL), `WATCHLIST_MATCH` (50) and `PEP_MATCH` (30).

## Scope notes

- Matching is name-based with birth-year filtering. Country matching is display-only: the lists give country names, customers carry ISO codes, and mapping them is out of scope.
- Customers get an optional date of birth so individual matches can be filtered by birth year. There is no customer update endpoint yet, so the date can only be set at creation.
- A list refresh rescreens every customer of every tenant. That is fine for MVP sizes (tens of thousands of customers); Phase 5's load test decides whether an index or delta screening is needed.
- A match is recorded once per customer and listed subject. Once reviewed (either way) it is not raised again, even if the list entry changes.
- Vessels and aircraft on the OFAC list are skipped; customers are individuals and businesses.
- UN "Low" quality aliases and OFAC "weak" aliases are skipped, as both publishers advise.

## Global Constraints

- Repos: `/Users/david/projects/money-laundry-detector/aegis` and `/Users/david/projects/money-laundry-detector/aegis-console`. Work on a `phase-3-screening` branch in each, off `main`.
- No new NuGet or npm packages. XML parsing, SHA-256, `IHttpClientFactory` and CSV handling use what is already in the framework or the repo.
- TDD: failing test first. Every new read or write endpoint gets a tenant isolation check.
- Every state change writes an audit event built with `AuditPayload.Json(...)`.
- Controller style stays as-is: `if (!_tenant.IsAuthenticated) return Unauthorized();` then `[RequirePermission(...)]`.
- Integration tests need Postgres (`docker-compose up -d postgres`, database `aegis_test`).
- Run commands from the repo root of the repo the task touches. Backend test command: `dotnet test tests/Unit` / `dotnet test tests/Integration`. Console: `npm test`, `npm run lint`, `npx tsc --noEmit`.
- Default list URLs (overridable in config):
  - UN: `https://scsanctions.un.org/resources/xml/en/consolidated.xml`
  - OFAC: `https://sanctionslistservice.ofac.treas.gov/api/PublicationPreview/exports/SDN.XML`
- Formats verified against the live files on 2026-10-04: UN root `CONSOLIDATED_LIST` (736 individuals, 275 entities; birth dates as `DATE`, `YEAR`, or `FROM_YEAR`/`TO_YEAR`); OFAC root `sdnList` in namespace `https://sanctionslistservice.ofac.treas.gov/api/PublicationPreview/exports/XML` (7,570 individuals, 10,037 entities; `dateOfBirth` like `10 Dec 1948`, `1948`, `circa 1948`, `1948 to 1950`).

## File map

**Backend (`aegis`)**

| File | Responsibility |
|---|---|
| `src/Aegis.Modules.Screening/Domain/WatchlistTypes.cs` | Enums and records shared by matching, parsing and storage |
| `src/Aegis.Modules.Screening/Domain/NameNormalizer.cs` | Name to comparable tokens |
| `src/Aegis.Modules.Screening/Domain/JaroWinkler.cs` | Token similarity |
| `src/Aegis.Modules.Screening/Domain/NameMatcher.cs` | Order-insensitive name score |
| `src/Aegis.Modules.Screening/Domain/WatchlistIndex.cs` | Screens one candidate against prepared subjects |
| `src/Aegis.Modules.Screening/Application/UnSanctionsParser.cs`, `OfacSdnParser.cs` | Official file parsers |
| `src/Aegis.Modules.Screening/Domain/WatchlistEntry.cs`, `WatchlistLoad.cs`, `TenantWatchlistEntry.cs`, `ScreeningMatch.cs` | Stored entities |
| `src/Aegis.Modules.Screening/Application/IScreeningRepositories.cs` | Repository ports |
| `src/Aegis.Application/Csv/CsvLine.cs` | CSV line splitting shared by both CSV parsers |
| `src/Aegis.Application/Screening/TenantWatchlistCsvParser.cs` | Tenant list CSV |
| `src/Aegis.Application/Screening/WatchlistRefresher.cs` | Download, verify, replace a global list |
| `src/Aegis.Application/Screening/CustomerScreeningService.cs` | Screen customers, review matches |
| `src/Aegis.Application/Screening/ScreeningAlertPolicy.cs` | Severity and evidence of screening alerts |
| `src/Aegis.Infrastructure/Screening/*` | Downloader, global list cache, candidate reader, refresh job, risk factor sync |
| `src/Aegis.Infrastructure/Persistence/Configurations/Screening/ScreeningConfigurations.cs` | EF mapping |
| `src/Aegis.Infrastructure/Persistence/Repositories/ScreeningRepositories.cs` | EF repositories |
| `src/Aegis.Api/Controllers/ScreeningController.cs` | HTTP API |
| `tests/Fixtures/screening/un-sample.xml`, `ofac-sample.xml` | Small files in the real formats, fictional names |

**Console (`aegis-console`)**

| File | Responsibility |
|---|---|
| `src/lib/screening/match.ts` (+ test) | Types, labels, CSV template |
| `src/lib/api/screening.ts` | API calls |
| `src/app/(app)/screening/page.tsx` | Review queue |
| `src/app/(app)/screening/[id]/page.tsx` | Match review |
| `src/app/(app)/admin/screening/page.tsx` | List status, refresh, tenant list upload |
| `src/components/screening/ScreeningPanel.tsx` | Customer page panel |

---

### Task 1: Matching core

**Files:**
- Delete: `src/Aegis.Modules.Screening/Class1.cs`
- Create: `src/Aegis.Modules.Screening/Domain/WatchlistTypes.cs`, `NameNormalizer.cs`, `JaroWinkler.cs`, `NameMatcher.cs`, `WatchlistIndex.cs`
- Modify: `tests/Unit/Aegis.Tests.Unit.csproj` (reference the Screening project)
- Test: `tests/Unit/Screening/NameMatchingTests.cs`, `tests/Unit/Screening/WatchlistIndexTests.cs`

**Interfaces:**
- Produces (namespace `Aegis.Modules.Screening.Domain`):
  - `enum WatchlistSource { UN, OFAC, TENANT }`, `enum WatchlistCategory { SANCTIONS, PEP, WATCHLIST }`, `enum SubjectKind { INDIVIDUAL, ENTITY }`
  - `record BirthYearRange(int From, int To)` with `bool Contains(int year, int tolerance)`
  - `record WatchlistSubject(WatchlistSource Source, string ExternalId, WatchlistCategory Category, SubjectKind Kind, IReadOnlyList<string> Names, IReadOnlyList<BirthYearRange> BirthYears, IReadOnlyList<string> Countries, string? Detail)`
  - `record ScreeningCandidate(Guid CustomerId, string Name, SubjectKind Kind, DateOnly? DateOfBirth)`
  - `record PotentialMatch(WatchlistSubject Subject, string MatchedName, double Score, bool BirthYearMatched)`
  - `static IReadOnlyList<string> NameNormalizer.Tokens(string? name)`
  - `static double JaroWinkler.Similarity(string a, string b)`
  - `static double NameMatcher.Score(IReadOnlyList<string> a, IReadOnlyList<string> b)`
  - `sealed class WatchlistIndex(IEnumerable<WatchlistSubject>)` with `int Count`, `IReadOnlyList<PotentialMatch> Screen(ScreeningCandidate candidate, double threshold = WatchlistIndex.DefaultThreshold)`, `const double DefaultThreshold = 0.88`, `const int BirthYearTolerance = 1`

- [ ] **Step 1: Reference the project and write the failing tests**

In `tests/Unit/Aegis.Tests.Unit.csproj`, add to the project-reference `ItemGroup`:

```xml
<ProjectReference Include="..\..\src\Aegis.Modules.Screening\Aegis.Modules.Screening.csproj" />
```

Create `tests/Unit/Screening/NameMatchingTests.cs`:

```csharp
namespace Aegis.Tests.Unit.Screening;

using Aegis.Modules.Screening.Domain;

public sealed class NameMatchingTests
{
    [Fact]
    public void Tokens_drop_diacritics_punctuation_and_titles()
        => Assert.Equal(new[] { "JOSE", "ALVAREZ", "NUNEZ" }, NameNormalizer.Tokens("Dr. José  Álvarez-Núñez"));

    [Fact]
    public void Tokens_drop_company_suffixes()
        => Assert.Equal(new[] { "BANCO", "NACIONAL", "DE", "CUBA" }, NameNormalizer.Tokens("Banco Nacional de Cuba Ltd."));

    [Fact]
    public void Tokens_keep_a_name_made_only_of_noise_words()
        => Assert.Equal(new[] { "THE", "COMPANY" }, NameNormalizer.Tokens("The Company"));

    [Fact]
    public void Tokens_of_blank_name_are_empty()
        => Assert.Empty(NameNormalizer.Tokens("  "));

    [Theory]
    [InlineData("MARTHA", "MARHTA", 0.961)]
    [InlineData("DWAYNE", "DUANE", 0.840)]
    [InlineData("DIXON", "DICKSONX", 0.813)]
    [InlineData("SAME", "SAME", 1.0)]
    [InlineData("ABC", "XYZ", 0.0)]
    public void Jaro_winkler_matches_reference_values(string a, string b, double expected)
        => Assert.Equal(expected, JaroWinkler.Similarity(a, b), 3);

    [Fact]
    public void Score_ignores_name_order()
        => Assert.Equal(1.0, Score("John Mwangi Kamau", "KAMAU John MWANGI"));

    [Fact]
    public void Missing_middle_name_still_scores_above_threshold()
        => Assert.Equal(0.9333, Score("John Kamau", "John Mwangi Kamau"), 4);

    [Fact]
    public void Transliteration_variants_score_above_threshold()
        => Assert.True(Score("Mohammed Ali", "Muhammad Ali") >= WatchlistIndex.DefaultThreshold);

    [Fact]
    public void Different_surname_scores_below_threshold()
        => Assert.True(Score("John Smith", "John Kamau") < 0.8);

    [Fact]
    public void One_shared_token_against_a_longer_name_is_capped_below_threshold()
        => Assert.True(Score("Saad", "Saad Omar Hassan") < WatchlistIndex.DefaultThreshold);

    private static double Score(string a, string b) => NameMatcher.Score(NameNormalizer.Tokens(a), NameNormalizer.Tokens(b));
}
```

Create `tests/Unit/Screening/WatchlistIndexTests.cs`:

```csharp
namespace Aegis.Tests.Unit.Screening;

using Aegis.Modules.Screening.Domain;

public sealed class WatchlistIndexTests
{
    private static readonly WatchlistSubject Listed = new(
        WatchlistSource.UN, "QDi.901", WatchlistCategory.SANCTIONS, SubjectKind.INDIVIDUAL,
        new[] { "ZEPHANIAH OKOTH MBWANA", "Zeph Mbwana" },
        new[] { new BirthYearRange(1978, 1978) },
        new[] { "Kenya" },
        "Al-Qaida");

    private static readonly WatchlistSubject Company = new(
        WatchlistSource.OFAC, "990002", WatchlistCategory.SANCTIONS, SubjectKind.ENTITY,
        new[] { "SILVERLINE LOGISTICS FZE" }, Array.Empty<BirthYearRange>(), Array.Empty<string>(), "IRAN");

    private readonly WatchlistIndex _index = new(new[] { Listed, Company });

    [Fact]
    public void Matches_on_alias_and_reports_the_best_name()
    {
        var match = Assert.Single(_index.Screen(Candidate("Zephaniah Mbwana")));
        Assert.Equal("QDi.901", match.Subject.ExternalId);
        Assert.True(match.Score >= WatchlistIndex.DefaultThreshold);
        Assert.False(match.BirthYearMatched);
    }

    [Fact]
    public void Birth_year_inside_tolerance_is_flagged()
        => Assert.True(Assert.Single(_index.Screen(Candidate("Zephaniah Mbwana", new DateOnly(1979, 3, 1)))).BirthYearMatched);

    [Fact]
    public void Birth_year_outside_the_listed_range_excludes_the_subject()
        => Assert.Empty(_index.Screen(Candidate("Zephaniah Mbwana", new DateOnly(1990, 1, 1))));

    [Fact]
    public void Kind_must_agree()
        => Assert.Empty(_index.Screen(new ScreeningCandidate(Guid.NewGuid(), "Zephaniah Mbwana", SubjectKind.ENTITY, null)));

    [Fact]
    public void Business_matches_entity_despite_suffix()
        => Assert.Equal("990002", Assert.Single(_index.Screen(
            new ScreeningCandidate(Guid.NewGuid(), "Silverline Logistics FZE Limited", SubjectKind.ENTITY, null))).Subject.ExternalId);

    [Fact]
    public void Unrelated_name_does_not_match()
        => Assert.Empty(_index.Screen(Candidate("Grace Akinyi")));

    private static ScreeningCandidate Candidate(string name, DateOnly? dob = null)
        => new(Guid.NewGuid(), name, SubjectKind.INDIVIDUAL, dob);
}
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `dotnet test tests/Unit --filter "FullyQualifiedName~Screening"`
Expected: build fails, `Aegis.Modules.Screening.Domain` types not found.

- [ ] **Step 3: Implement the matching core**

Delete `src/Aegis.Modules.Screening/Class1.cs`.

Create `src/Aegis.Modules.Screening/Domain/WatchlistTypes.cs`:

```csharp
namespace Aegis.Modules.Screening.Domain;

public enum WatchlistSource { UN, OFAC, TENANT }

public enum WatchlistCategory { SANCTIONS, PEP, WATCHLIST }

public enum SubjectKind { INDIVIDUAL, ENTITY }

/// <summary>Inclusive range of possible birth years; lists often give only a year or a span.</summary>
public sealed record BirthYearRange(int From, int To)
{
    public bool Contains(int year, int tolerance) => year >= From - tolerance && year <= To + tolerance;
}

/// <summary>One listed person or organisation. <see cref="Names"/>[0] is the primary name; the rest are aliases.</summary>
public sealed record WatchlistSubject(
    WatchlistSource Source,
    string ExternalId,
    WatchlistCategory Category,
    SubjectKind Kind,
    IReadOnlyList<string> Names,
    IReadOnlyList<BirthYearRange> BirthYears,
    IReadOnlyList<string> Countries,
    string? Detail);

/// <summary>The customer being screened. Businesses screen as <see cref="SubjectKind.ENTITY"/>.</summary>
public sealed record ScreeningCandidate(Guid CustomerId, string Name, SubjectKind Kind, DateOnly? DateOfBirth);

public sealed record PotentialMatch(WatchlistSubject Subject, string MatchedName, double Score, bool BirthYearMatched);
```

Create `src/Aegis.Modules.Screening/Domain/NameNormalizer.cs`:

```csharp
namespace Aegis.Modules.Screening.Domain;

using System.Globalization;
using System.Text;

public static class NameNormalizer
{
    private static readonly HashSet<string> Noise = new(StringComparer.Ordinal)
    {
        "MR", "MRS", "MS", "MISS", "DR", "PROF", "SIR", "HON", "ENG", "REV",
        "LTD", "LIMITED", "CO", "COMPANY", "INC", "LLC", "PLC", "CORP", "CORPORATION", "SA", "GMBH", "THE"
    };

    /// <summary>Upper-case tokens without diacritics, punctuation, titles or company suffixes.</summary>
    public static IReadOnlyList<string> Tokens(string? name)
    {
        if (string.IsNullOrWhiteSpace(name)) return Array.Empty<string>();

        var builder = new StringBuilder(name.Length);
        foreach (var ch in name.Normalize(NormalizationForm.FormD))
        {
            if (CharUnicodeInfo.GetUnicodeCategory(ch) == UnicodeCategory.NonSpacingMark) continue;
            builder.Append(char.IsLetterOrDigit(ch) ? char.ToUpperInvariant(ch) : ' ');
        }

        var tokens = builder.ToString().Split(' ', StringSplitOptions.RemoveEmptyEntries);
        var kept = tokens.Where(t => !Noise.Contains(t)).ToArray();
        return kept.Length > 0 ? kept : tokens;
    }
}
```

Create `src/Aegis.Modules.Screening/Domain/JaroWinkler.cs`:

```csharp
namespace Aegis.Modules.Screening.Domain;

public static class JaroWinkler
{
    public static double Similarity(string a, string b)
    {
        if (a == b) return 1;
        if (a.Length == 0 || b.Length == 0) return 0;

        var range = Math.Max(0, Math.Max(a.Length, b.Length) / 2 - 1);
        var aMatched = new bool[a.Length];
        var bMatched = new bool[b.Length];
        var matches = 0;
        for (var i = 0; i < a.Length; i++)
        {
            var end = Math.Min(b.Length - 1, i + range);
            for (var j = Math.Max(0, i - range); j <= end; j++)
            {
                if (bMatched[j] || a[i] != b[j]) continue;
                aMatched[i] = bMatched[j] = true;
                matches++;
                break;
            }
        }
        if (matches == 0) return 0;

        var transpositions = 0;
        for (int i = 0, k = 0; i < a.Length; i++)
        {
            if (!aMatched[i]) continue;
            while (!bMatched[k]) k++;
            if (a[i] != b[k]) transpositions++;
            k++;
        }

        var m = (double)matches;
        var jaro = (m / a.Length + m / b.Length + (m - transpositions / 2.0) / m) / 3;
        var prefix = 0;
        var maxPrefix = Math.Min(4, Math.Min(a.Length, b.Length));
        while (prefix < maxPrefix && a[prefix] == b[prefix]) prefix++;
        return jaro + prefix * 0.1 * (1 - jaro);
    }
}
```

Create `src/Aegis.Modules.Screening/Domain/NameMatcher.cs`:

```csharp
namespace Aegis.Modules.Screening.Domain;

public static class NameMatcher
{
    /// <summary>A single shared token is too weak to call a match against a longer name.</summary>
    private const double SingleTokenCap = 0.85;

    /// <summary>
    /// Order-insensitive similarity in [0, 1]. Each token of the shorter name is paired with its best unused
    /// token in the longer name; the average pair similarity is reduced when the longer name has extra tokens.
    /// </summary>
    public static double Score(IReadOnlyList<string> a, IReadOnlyList<string> b)
    {
        if (a.Count == 0 || b.Count == 0) return 0;
        var (shorter, longer) = a.Count <= b.Count ? (a, b) : (b, a);

        var pairs = new List<(int S, int L, double Sim)>(shorter.Count * longer.Count);
        for (var s = 0; s < shorter.Count; s++)
            for (var l = 0; l < longer.Count; l++)
                pairs.Add((s, l, JaroWinkler.Similarity(shorter[s], longer[l])));

        var usedS = new bool[shorter.Count];
        var usedL = new bool[longer.Count];
        var total = 0.0;
        foreach (var p in pairs.OrderByDescending(p => p.Sim))
        {
            if (usedS[p.S] || usedL[p.L]) continue;
            usedS[p.S] = usedL[p.L] = true;
            total += p.Sim;
        }

        var coverage = (double)shorter.Count / longer.Count;
        var score = Math.Round(total / shorter.Count * (0.8 + 0.2 * coverage), 4);
        return shorter.Count == 1 && longer.Count > 1 ? Math.Min(score, SingleTokenCap) : score;
    }
}
```

Create `src/Aegis.Modules.Screening/Domain/WatchlistIndex.cs`:

```csharp
namespace Aegis.Modules.Screening.Domain;

/// <summary>Subjects with their names tokenized once, so many customers can be screened cheaply.</summary>
public sealed class WatchlistIndex
{
    public const double DefaultThreshold = 0.88;

    /// <summary>A customer born this many years outside the listed range is still treated as consistent.</summary>
    public const int BirthYearTolerance = 1;

    private readonly List<(WatchlistSubject Subject, (string Name, IReadOnlyList<string> Tokens)[] Names)> _entries;

    public WatchlistIndex(IEnumerable<WatchlistSubject> subjects)
        => _entries = subjects
            .Select(s => (s, s.Names.Select(n => (n, NameNormalizer.Tokens(n))).ToArray()))
            .ToList();

    public int Count => _entries.Count;

    public IReadOnlyList<PotentialMatch> Screen(ScreeningCandidate candidate, double threshold = DefaultThreshold)
    {
        var tokens = NameNormalizer.Tokens(candidate.Name);
        if (tokens.Count == 0) return Array.Empty<PotentialMatch>();

        var matches = new List<PotentialMatch>();
        foreach (var (subject, names) in _entries)
        {
            if (subject.Kind != candidate.Kind) continue;

            var birthYearMatched = false;
            if (candidate.DateOfBirth is { } dob && subject.BirthYears.Count > 0)
            {
                birthYearMatched = subject.BirthYears.Any(r => r.Contains(dob.Year, BirthYearTolerance));
                if (!birthYearMatched) continue;
            }

            var best = 0.0;
            string? bestName = null;
            foreach (var (name, nameTokens) in names)
            {
                var score = NameMatcher.Score(tokens, nameTokens);
                if (score > best)
                {
                    best = score;
                    bestName = name;
                }
            }

            if (bestName is not null && best >= threshold)
                matches.Add(new PotentialMatch(subject, bestName, best, birthYearMatched));
        }

        return matches.OrderByDescending(m => m.Score).ToList();
    }
}
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `dotnet test tests/Unit --filter "FullyQualifiedName~Screening"`
Expected: all pass. Then `dotnet test tests/Unit`: no regressions.

- [ ] **Step 5: Commit**

```bash
git add src/Aegis.Modules.Screening tests/Unit/Aegis.Tests.Unit.csproj tests/Unit/Screening
git commit -m "feat(screening): name normalizer, Jaro-Winkler and watchlist index"
```

---

### Task 2: UN and OFAC parsers

**Files:**
- Create: `src/Aegis.Modules.Screening/Application/WatchlistParseResult.cs`, `UnSanctionsParser.cs`, `OfacSdnParser.cs`
- Create: `tests/Fixtures/screening/un-sample.xml`, `tests/Fixtures/screening/ofac-sample.xml`
- Modify: `tests/Unit/Aegis.Tests.Unit.csproj`, `tests/Integration/Aegis.Tests.Integration.csproj` (copy fixtures to output)
- Test: `tests/Unit/Screening/ListParserTests.cs`

**Interfaces:**
- Consumes: Task 1 types.
- Produces (namespace `Aegis.Modules.Screening.Application`):
  - `record WatchlistParseResult(IReadOnlyList<WatchlistSubject> Subjects, DateTimeOffset? PublishedAt)`
  - `static WatchlistParseResult UnSanctionsParser.Parse(Stream xml)` (throws `FormatException` for a wrong root)
  - `static WatchlistParseResult OfacSdnParser.Parse(Stream xml)` (same)
  - `static BirthYearRange? OfacSdnParser.YearRange(string? text)`
- Fixture paths at test runtime: `Path.Combine(AppContext.BaseDirectory, "Fixtures", "screening", "un-sample.xml")` and `"ofac-sample.xml"`.

- [ ] **Step 1: Add the fixtures**

Create `tests/Fixtures/screening/un-sample.xml` (real structure, fictional people):

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<CONSOLIDATED_LIST dateGenerated="2026-10-03T23:00:03.428Z">
  <INDIVIDUALS>
    <INDIVIDUAL>
      <DATAID>9000001</DATAID>
      <FIRST_NAME>ZEPHANIAH</FIRST_NAME>
      <SECOND_NAME>OKOTH</SECOND_NAME>
      <THIRD_NAME>MBWANA</THIRD_NAME>
      <UN_LIST_TYPE>Al-Qaida</UN_LIST_TYPE>
      <REFERENCE_NUMBER>QDi.901</REFERENCE_NUMBER>
      <LISTED_ON>2014-05-20</LISTED_ON>
      <NATIONALITY><VALUE>Kenya</VALUE></NATIONALITY>
      <INDIVIDUAL_ALIAS><QUALITY>Good</QUALITY><ALIAS_NAME>Zeph Mbwana</ALIAS_NAME></INDIVIDUAL_ALIAS>
      <INDIVIDUAL_ALIAS><QUALITY>Low</QUALITY><ALIAS_NAME>Abu Z</ALIAS_NAME></INDIVIDUAL_ALIAS>
      <INDIVIDUAL_DATE_OF_BIRTH><TYPE_OF_DATE>EXACT</TYPE_OF_DATE><DATE>1978-04-28</DATE></INDIVIDUAL_DATE_OF_BIRTH>
    </INDIVIDUAL>
    <INDIVIDUAL>
      <DATAID>9000002</DATAID>
      <FIRST_NAME>BARAKA</FIRST_NAME>
      <SECOND_NAME>TUMAINI</SECOND_NAME>
      <UN_LIST_TYPE>Somalia</UN_LIST_TYPE>
      <REFERENCE_NUMBER>SOi.902</REFERENCE_NUMBER>
      <LISTED_ON>2018-11-01</LISTED_ON>
      <INDIVIDUAL_ALIAS><QUALITY/><ALIAS_NAME/></INDIVIDUAL_ALIAS>
      <INDIVIDUAL_DATE_OF_BIRTH><TYPE_OF_DATE>BETWEEN</TYPE_OF_DATE><FROM_YEAR>1973</FROM_YEAR><TO_YEAR>1975</TO_YEAR></INDIVIDUAL_DATE_OF_BIRTH>
    </INDIVIDUAL>
  </INDIVIDUALS>
  <ENTITIES>
    <ENTITY>
      <DATAID>9000101</DATAID>
      <FIRST_NAME>CORAL REEF TRADING COMPANY</FIRST_NAME>
      <UN_LIST_TYPE>Al-Qaida</UN_LIST_TYPE>
      <REFERENCE_NUMBER>QDe.951</REFERENCE_NUMBER>
      <LISTED_ON>2016-02-10</LISTED_ON>
      <ENTITY_ALIAS><QUALITY>a.k.a.</QUALITY><ALIAS_NAME>Coral Reef General Traders</ALIAS_NAME></ENTITY_ALIAS>
    </ENTITY>
  </ENTITIES>
</CONSOLIDATED_LIST>
```

Create `tests/Fixtures/screening/ofac-sample.xml`:

```xml
<?xml version="1.0" standalone="yes"?>
<sdnList xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns="https://sanctionslistservice.ofac.treas.gov/api/PublicationPreview/exports/XML">
  <publshInformation><Publish_Date>10/02/2026</Publish_Date><Record_Count>3</Record_Count></publshInformation>
  <sdnEntry>
    <uid>990001</uid>
    <firstName>Amina Wanjiru</firstName>
    <lastName>NDEGWA</lastName>
    <sdnType>Individual</sdnType>
    <programList><program>SDGT</program></programList>
    <akaList>
      <aka><uid>1</uid><type>a.k.a.</type><category>strong</category><firstName>Amina</firstName><lastName>NDEGWA-ODHIAMBO</lastName></aka>
      <aka><uid>2</uid><type>a.k.a.</type><category>weak</category><lastName>MAMA A</lastName></aka>
    </akaList>
    <nationalityList><nationality><uid>3</uid><country>Kenya</country><mainEntry>true</mainEntry></nationality></nationalityList>
    <dateOfBirthList><dateOfBirthItem><uid>4</uid><dateOfBirth>10 Dec 1968</dateOfBirth><mainEntry>true</mainEntry></dateOfBirthItem></dateOfBirthList>
  </sdnEntry>
  <sdnEntry>
    <uid>990002</uid>
    <lastName>SILVERLINE LOGISTICS FZE</lastName>
    <sdnType>Entity</sdnType>
    <programList><program>IRAN</program><program>NPWMD</program></programList>
  </sdnEntry>
  <sdnEntry>
    <uid>990003</uid>
    <lastName>OCEAN STAR</lastName>
    <sdnType>Vessel</sdnType>
  </sdnEntry>
</sdnList>
```

Add to both `tests/Unit/Aegis.Tests.Unit.csproj` and `tests/Integration/Aegis.Tests.Integration.csproj`:

```xml
<ItemGroup>
  <None Include="..\Fixtures\screening\*.xml" LinkBase="Fixtures\screening" CopyToOutputDirectory="PreserveNewest" />
</ItemGroup>
```

- [ ] **Step 2: Write the failing tests**

Create `tests/Unit/Screening/ListParserTests.cs`:

```csharp
namespace Aegis.Tests.Unit.Screening;

using System.Text;
using Aegis.Modules.Screening.Application;
using Aegis.Modules.Screening.Domain;

public sealed class ListParserTests
{
    private static Stream Fixture(string name)
        => File.OpenRead(Path.Combine(AppContext.BaseDirectory, "Fixtures", "screening", name));

    [Fact]
    public void Un_list_parses_individuals_and_entities()
    {
        var result = UnSanctionsParser.Parse(Fixture("un-sample.xml"));

        Assert.Equal(DateTimeOffset.Parse("2026-10-03T23:00:03.428Z"), result.PublishedAt);
        Assert.Equal(3, result.Subjects.Count);

        var zeph = result.Subjects.Single(s => s.ExternalId == "QDi.901");
        Assert.Equal(WatchlistSource.UN, zeph.Source);
        Assert.Equal(WatchlistCategory.SANCTIONS, zeph.Category);
        Assert.Equal(SubjectKind.INDIVIDUAL, zeph.Kind);
        Assert.Equal(new[] { "ZEPHANIAH OKOTH MBWANA", "Zeph Mbwana" }, zeph.Names);
        Assert.Equal(new[] { new BirthYearRange(1978, 1978) }, zeph.BirthYears);
        Assert.Equal(new[] { "Kenya" }, zeph.Countries);
        Assert.Equal("Al-Qaida · listed 2014-05-20", zeph.Detail);

        var baraka = result.Subjects.Single(s => s.ExternalId == "SOi.902");
        Assert.Equal(new[] { "BARAKA TUMAINI" }, baraka.Names);
        Assert.Equal(new[] { new BirthYearRange(1973, 1975) }, baraka.BirthYears);

        var entity = result.Subjects.Single(s => s.ExternalId == "QDe.951");
        Assert.Equal(SubjectKind.ENTITY, entity.Kind);
        Assert.Equal(new[] { "CORAL REEF TRADING COMPANY", "Coral Reef General Traders" }, entity.Names);
    }

    [Fact]
    public void Ofac_list_parses_individuals_and_entities_and_skips_vessels()
    {
        var result = OfacSdnParser.Parse(Fixture("ofac-sample.xml"));

        Assert.Equal(new DateTimeOffset(2026, 10, 2, 0, 0, 0, TimeSpan.Zero), result.PublishedAt);
        Assert.Equal(2, result.Subjects.Count);

        var amina = result.Subjects.Single(s => s.ExternalId == "990001");
        Assert.Equal(WatchlistSource.OFAC, amina.Source);
        Assert.Equal(SubjectKind.INDIVIDUAL, amina.Kind);
        Assert.Equal(new[] { "Amina Wanjiru NDEGWA", "Amina NDEGWA-ODHIAMBO" }, amina.Names);
        Assert.Equal(new[] { new BirthYearRange(1968, 1968) }, amina.BirthYears);
        Assert.Equal(new[] { "Kenya" }, amina.Countries);
        Assert.Equal("SDGT", amina.Detail);

        var company = result.Subjects.Single(s => s.ExternalId == "990002");
        Assert.Equal(SubjectKind.ENTITY, company.Kind);
        Assert.Equal("IRAN, NPWMD", company.Detail);
    }

    [Theory]
    [InlineData("10 Dec 1948", 1948, 1948)]
    [InlineData("1948", 1948, 1948)]
    [InlineData("circa 1960", 1960, 1960)]
    [InlineData("1948 to 1950", 1948, 1950)]
    public void Ofac_birth_dates_become_year_ranges(string text, int from, int to)
        => Assert.Equal(new BirthYearRange(from, to), OfacSdnParser.YearRange(text));

    [Fact]
    public void Ofac_birth_date_without_a_year_is_ignored()
        => Assert.Null(OfacSdnParser.YearRange("unknown"));

    [Fact]
    public void Wrong_root_element_is_rejected()
    {
        var xml = new MemoryStream(Encoding.UTF8.GetBytes("<html></html>"));
        Assert.Throws<FormatException>(() => UnSanctionsParser.Parse(xml));
        xml.Position = 0;
        Assert.Throws<FormatException>(() => OfacSdnParser.Parse(xml));
    }
}
```

- [ ] **Step 3: Run the tests to verify they fail**

Run: `dotnet test tests/Unit --filter "FullyQualifiedName~ListParserTests"`
Expected: build fails, parsers not found.

- [ ] **Step 4: Implement the parsers**

`XDocument.Load` uses `DtdProcessing.Prohibit` by default, so external entities in a tampered file are rejected.

Create `src/Aegis.Modules.Screening/Application/WatchlistParseResult.cs`:

```csharp
namespace Aegis.Modules.Screening.Application;

using Aegis.Modules.Screening.Domain;

public sealed record WatchlistParseResult(IReadOnlyList<WatchlistSubject> Subjects, DateTimeOffset? PublishedAt);
```

Create `src/Aegis.Modules.Screening/Application/UnSanctionsParser.cs`:

```csharp
namespace Aegis.Modules.Screening.Application;

using System.Globalization;
using System.Xml.Linq;
using Aegis.Modules.Screening.Domain;

/// <summary>UN Security Council consolidated list (scsanctions.un.org consolidated.xml).</summary>
public static class UnSanctionsParser
{
    public static WatchlistParseResult Parse(Stream xml)
    {
        var root = XDocument.Load(xml).Root;
        if (root is null || root.Name.LocalName != "CONSOLIDATED_LIST")
            throw new FormatException("Not a UN consolidated list.");

        DateTimeOffset? published = DateTimeOffset.TryParse(
            (string?)root.Attribute("dateGenerated"), CultureInfo.InvariantCulture, DateTimeStyles.AssumeUniversal, out var p)
            ? p
            : null;

        var subjects = new List<WatchlistSubject>();
        foreach (var e in root.Element("INDIVIDUALS")?.Elements("INDIVIDUAL") ?? Enumerable.Empty<XElement>())
        {
            var primary = Join(e, "FIRST_NAME", "SECOND_NAME", "THIRD_NAME", "FOURTH_NAME");
            if (Build(e, SubjectKind.INDIVIDUAL, primary, "INDIVIDUAL_ALIAS", BirthYears(e), Values(e.Element("NATIONALITY"))) is { } s)
                subjects.Add(s);
        }
        foreach (var e in root.Element("ENTITIES")?.Elements("ENTITY") ?? Enumerable.Empty<XElement>())
        {
            if (Build(e, SubjectKind.ENTITY, Text(e, "FIRST_NAME"), "ENTITY_ALIAS", Array.Empty<BirthYearRange>(), Array.Empty<string>()) is { } s)
                subjects.Add(s);
        }

        return new WatchlistParseResult(subjects, published);
    }

    private static WatchlistSubject? Build(
        XElement e, SubjectKind kind, string? primary, string aliasElement,
        IReadOnlyList<BirthYearRange> birthYears, IReadOnlyList<string> countries)
    {
        var reference = Text(e, "REFERENCE_NUMBER") ?? Text(e, "DATAID");
        if (primary is null || reference is null) return null;

        var names = new List<string> { primary };
        foreach (var alias in e.Elements(aliasElement))
        {
            // The UN advises against screening on "Low" quality aliases.
            if (string.Equals(Text(alias, "QUALITY"), "Low", StringComparison.OrdinalIgnoreCase)) continue;
            if (Text(alias, "ALIAS_NAME") is { } name && !names.Contains(name, StringComparer.OrdinalIgnoreCase))
                names.Add(name);
        }

        var detailParts = new[] { Text(e, "UN_LIST_TYPE"), Text(e, "LISTED_ON") is { } on ? $"listed {on}" : null };
        var detail = string.Join(" · ", detailParts.Where(d => d is not null));
        return new WatchlistSubject(
            WatchlistSource.UN, reference, WatchlistCategory.SANCTIONS, kind, names, birthYears, countries,
            detail.Length == 0 ? null : detail);
    }

    private static IReadOnlyList<BirthYearRange> BirthYears(XElement e)
    {
        var ranges = new List<BirthYearRange>();
        foreach (var d in e.Elements("INDIVIDUAL_DATE_OF_BIRTH"))
        {
            if (DateOnly.TryParse(Text(d, "DATE"), CultureInfo.InvariantCulture, out var date))
                ranges.Add(new BirthYearRange(date.Year, date.Year));
            else if (int.TryParse(Text(d, "YEAR"), CultureInfo.InvariantCulture, out var year))
                ranges.Add(new BirthYearRange(year, year));
            else if (int.TryParse(Text(d, "FROM_YEAR"), CultureInfo.InvariantCulture, out var from)
                     && int.TryParse(Text(d, "TO_YEAR"), CultureInfo.InvariantCulture, out var to))
                ranges.Add(new BirthYearRange(Math.Min(from, to), Math.Max(from, to)));
        }
        return ranges;
    }

    private static string? Text(XElement e, string name)
    {
        var value = e.Element(name)?.Value.Trim();
        return string.IsNullOrEmpty(value) ? null : value;
    }

    private static string? Join(XElement e, params string[] parts)
    {
        var joined = string.Join(" ", parts.Select(p => Text(e, p)).Where(v => v is not null));
        return joined.Length == 0 ? null : joined;
    }

    private static IReadOnlyList<string> Values(XElement? e)
        => e?.Elements("VALUE").Select(v => v.Value.Trim()).Where(v => v.Length > 0).ToList() ?? new List<string>();
}
```

Create `src/Aegis.Modules.Screening/Application/OfacSdnParser.cs`:

```csharp
namespace Aegis.Modules.Screening.Application;

using System.Globalization;
using System.Text.RegularExpressions;
using System.Xml.Linq;
using Aegis.Modules.Screening.Domain;

/// <summary>US Treasury OFAC Specially Designated Nationals list (SDN.XML).</summary>
public static class OfacSdnParser
{
    private static readonly Regex Year = new(@"\b(1[89]\d{2}|20\d{2})\b", RegexOptions.Compiled);

    public static WatchlistParseResult Parse(Stream xml)
    {
        var root = XDocument.Load(xml).Root;
        if (root is null || root.Name.LocalName != "sdnList")
            throw new FormatException("Not an OFAC SDN list.");
        var ns = root.Name.Namespace;

        string? Text(XElement e, string name)
        {
            var value = e.Element(ns + name)?.Value.Trim();
            return string.IsNullOrEmpty(value) ? null : value;
        }

        string? Name(XElement e)
        {
            var name = string.Join(" ", new[] { Text(e, "firstName"), Text(e, "lastName") }.Where(v => v is not null));
            return name.Length == 0 ? null : name;
        }

        IEnumerable<XElement> Items(XElement e, string list, string item)
            => e.Element(ns + list)?.Elements(ns + item) ?? Enumerable.Empty<XElement>();

        var publishInfo = root.Element(ns + "publshInformation");
        DateTimeOffset? published = DateTime.TryParseExact(
            publishInfo?.Element(ns + "Publish_Date")?.Value, "MM/dd/yyyy", CultureInfo.InvariantCulture,
            DateTimeStyles.AssumeUniversal | DateTimeStyles.AdjustToUniversal, out var p)
            ? new DateTimeOffset(p, TimeSpan.Zero)
            : null;

        var subjects = new List<WatchlistSubject>();
        foreach (var e in root.Elements(ns + "sdnEntry"))
        {
            SubjectKind? kind = Text(e, "sdnType") switch
            {
                "Individual" => SubjectKind.INDIVIDUAL,
                "Entity" => SubjectKind.ENTITY,
                _ => null
            };
            var uid = Text(e, "uid");
            var primary = Name(e);
            if (kind is null || uid is null || primary is null) continue;

            var names = new List<string> { primary };
            foreach (var aka in Items(e, "akaList", "aka"))
            {
                // OFAC marks low-quality aliases as "weak" and advises against screening on them.
                if (string.Equals(Text(aka, "category"), "weak", StringComparison.OrdinalIgnoreCase)) continue;
                if (Name(aka) is { } alias && !names.Contains(alias, StringComparer.OrdinalIgnoreCase)) names.Add(alias);
            }

            var birthYears = Items(e, "dateOfBirthList", "dateOfBirthItem")
                .Select(d => YearRange(Text(d, "dateOfBirth")))
                .OfType<BirthYearRange>()
                .ToList();
            var countries = Items(e, "nationalityList", "nationality")
                .Concat(Items(e, "citizenshipList", "citizenship"))
                .Select(n => Text(n, "country"))
                .OfType<string>()
                .Distinct()
                .ToList();
            var programs = Items(e, "programList", "program").Select(x => x.Value.Trim()).Where(x => x.Length > 0).ToList();

            subjects.Add(new WatchlistSubject(
                WatchlistSource.OFAC, uid, WatchlistCategory.SANCTIONS, kind.Value, names, birthYears, countries,
                programs.Count == 0 ? null : string.Join(", ", programs)));
        }

        return new WatchlistParseResult(subjects, published);
    }

    /// <summary>"10 Dec 1948", "circa 1948" and "1948 to 1950" all become year ranges; text without a year is null.</summary>
    public static BirthYearRange? YearRange(string? text)
    {
        if (text is null) return null;
        var years = Year.Matches(text).Select(m => int.Parse(m.Value, CultureInfo.InvariantCulture)).ToList();
        return years.Count == 0 ? null : new BirthYearRange(years.Min(), years.Max());
    }
}
```

- [ ] **Step 5: Run the tests to verify they pass**

Run: `dotnet test tests/Unit --filter "FullyQualifiedName~ListParserTests"`
Expected: PASS.

- [ ] **Step 6: Smoke-test against the live files (manual, not committed)**

```bash
curl -sSL -o /tmp/un.xml https://scsanctions.un.org/resources/xml/en/consolidated.xml
curl -sSL -o /tmp/sdn.xml https://sanctionslistservice.ofac.treas.gov/api/PublicationPreview/exports/SDN.XML
```

Temporarily add a `[Fact(Skip = "manual")]` test (or use `dotnet script` if installed) that parses both files and prints the counts. Expected: about 1,000 UN subjects and about 17,600 OFAC subjects (individuals plus entities). Don't commit the temporary test.

- [ ] **Step 7: Commit**

```bash
git add src/Aegis.Modules.Screening/Application tests/Fixtures tests/Unit tests/Integration/Aegis.Tests.Integration.csproj
git commit -m "feat(screening): UN consolidated and OFAC SDN list parsers"
```

---

### Task 3: Tenant list CSV parser

**Files:**
- Create: `src/Aegis.Application/Csv/CsvLine.cs`
- Modify: `src/Aegis.Application/Transactions/TransactionCsvParser.cs` (use `CsvLine`, delete its private `SplitLine`)
- Modify: `src/Aegis.Application/Aegis.Application.csproj` (reference Screening)
- Create: `src/Aegis.Application/Screening/TenantWatchlistCsvParser.cs`
- Test: `tests/Unit/Screening/TenantWatchlistCsvParserTests.cs` (existing `TransactionCsvParser` tests cover the refactor)

**Interfaces:**
- Produces:
  - `static List<string>? CsvLine.Split(string line)`; `static string[] CsvLine.Lines(string csv)` (namespace `Aegis.Application.Csv`)
  - `record TenantWatchlistParseResult(IReadOnlyList<string> Errors, IReadOnlyList<WatchlistSubject> Subjects)`
  - `static TenantWatchlistParseResult TenantWatchlistCsvParser.Parse(string csv)`; `const int MaxRows = 10_000`
  - CSV columns: required `reference,name,category,kind`; optional `aliases` (`;`-separated), `birthYear` (`1970` or `1968-1972`), `country`, `notes`.

- [ ] **Step 1: Write the failing tests**

Create `tests/Unit/Screening/TenantWatchlistCsvParserTests.cs`:

```csharp
namespace Aegis.Tests.Unit.Screening;

using Aegis.Application.Screening;
using Aegis.Modules.Screening.Domain;

public sealed class TenantWatchlistCsvParserTests
{
    private const string Header = "reference,name,category,kind,aliases,birthYear,country,notes\n";

    [Fact]
    public void Parses_rows_with_aliases_ranges_and_quoted_names()
    {
        var result = TenantWatchlistCsvParser.Parse(Header +
            "PEP-1,\"Wanjiku, Jane Doe\",pep,individual,Jane W. Doe; J. Wanjiku,1968-1970,KE,Former governor\n" +
            "INT-7,Blue Lagoon Traders,WATCHLIST,ENTITY,,,,\n");

        Assert.Empty(result.Errors);
        Assert.Equal(2, result.Subjects.Count);
        var pep = result.Subjects[0];
        Assert.Equal(WatchlistSource.TENANT, pep.Source);
        Assert.Equal("PEP-1", pep.ExternalId);
        Assert.Equal(WatchlistCategory.PEP, pep.Category);
        Assert.Equal(SubjectKind.INDIVIDUAL, pep.Kind);
        Assert.Equal(new[] { "Wanjiku, Jane Doe", "Jane W. Doe", "J. Wanjiku" }, pep.Names);
        Assert.Equal(new[] { new BirthYearRange(1968, 1970) }, pep.BirthYears);
        Assert.Equal(new[] { "KE" }, pep.Countries);
        Assert.Equal("Former governor", pep.Detail);
        Assert.Empty(result.Subjects[1].BirthYears);
    }

    [Fact]
    public void Missing_required_column_is_a_header_error()
    {
        var result = TenantWatchlistCsvParser.Parse("reference,name,kind\nA,B,INDIVIDUAL\n");
        Assert.Equal("Missing required columns: category.", Assert.Single(result.Errors));
        Assert.Empty(result.Subjects);
    }

    [Fact]
    public void Any_bad_row_rejects_the_whole_file()
    {
        var result = TenantWatchlistCsvParser.Parse(Header +
            "OK-1,Good Row,PEP,INDIVIDUAL,,,,\n" +
            "BAD-1,Bad Row,FRIEND,PERSON,,19x0,,\n" +
            "OK-1,Duplicate,PEP,INDIVIDUAL,,,,\n");

        Assert.Empty(result.Subjects);
        Assert.Equal(2, result.Errors.Count);
        Assert.Equal(
            "Line 3: category must be PEP, SANCTIONS or WATCHLIST; kind must be INDIVIDUAL or ENTITY; birthYear must be a year (1970) or a range (1968-1972).",
            result.Errors[0]);
        Assert.Equal("Line 4: reference 'OK-1' appears more than once.", result.Errors[1]);
    }

    [Fact]
    public void File_with_only_a_header_is_rejected()
        => Assert.Equal("The file has no rows.", Assert.Single(TenantWatchlistCsvParser.Parse(Header).Errors));
}
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `dotnet test tests/Unit --filter "FullyQualifiedName~TenantWatchlistCsvParserTests"`
Expected: build fails, `Aegis.Application.Screening` not found.

- [ ] **Step 3: Extract `CsvLine` and implement the parser**

Add to `src/Aegis.Application/Aegis.Application.csproj`:

```xml
<ProjectReference Include="..\Aegis.Modules.Screening\Aegis.Modules.Screening.csproj" />
```

Create `src/Aegis.Application/Csv/CsvLine.cs` by moving the body of `TransactionCsvParser.SplitLine` unchanged:

```csharp
namespace Aegis.Application.Csv;

using System.Text;

/// <summary>RFC 4180 quoting within one physical line (records never span lines).</summary>
public static class CsvLine
{
    public static string[] Lines(string csv) => csv.Replace("\r\n", "\n").Replace('\r', '\n').Split('\n');

    /// <returns>Null when a quoted field is not terminated on the same line.</returns>
    public static List<string>? Split(string line)
    {
        var fields = new List<string>();
        var current = new StringBuilder();
        var inQuotes = false;

        for (var i = 0; i < line.Length; i++)
        {
            var ch = line[i];
            if (inQuotes)
            {
                if (ch == '"' && i + 1 < line.Length && line[i + 1] == '"')
                {
                    current.Append('"');
                    i++;
                }
                else if (ch == '"')
                {
                    inQuotes = false;
                }
                else
                {
                    current.Append(ch);
                }
            }
            else if (ch == '"')
            {
                inQuotes = true;
            }
            else if (ch == ',')
            {
                fields.Add(current.ToString());
                current.Clear();
            }
            else
            {
                current.Append(ch);
            }
        }

        if (inQuotes) return null;
        fields.Add(current.ToString());
        return fields;
    }
}
```

In `TransactionCsvParser.cs`: add `using Aegis.Application.Csv;`, replace `csv.Replace("\r\n", "\n").Replace('\r', '\n').Split('\n')` with `CsvLine.Lines(csv)`, replace both `SplitLine(` calls with `CsvLine.Split(`, delete the private `SplitLine` method and the now-unused `using System.Text;`.

Create `src/Aegis.Application/Screening/TenantWatchlistCsvParser.cs`:

```csharp
namespace Aegis.Application.Screening;

using System.Globalization;
using Aegis.Application.Csv;
using Aegis.Modules.Screening.Domain;

public sealed record TenantWatchlistParseResult(IReadOnlyList<string> Errors, IReadOnlyList<WatchlistSubject> Subjects);

/// <summary>
/// The institution's own list (PEPs, the Kenya domestic list, internal blacklists). All-or-nothing: any row
/// error rejects the file, so a list is never half-replaced.
/// </summary>
public static class TenantWatchlistCsvParser
{
    public const int MaxRows = 10_000;

    private static readonly string[] Required = { "reference", "name", "category", "kind" };

    public static TenantWatchlistParseResult Parse(string csv)
    {
        var lines = CsvLine.Lines(csv);
        var headerIndex = Array.FindIndex(lines, l => !string.IsNullOrWhiteSpace(l));
        if (headerIndex < 0) return Fail("CSV is empty.");

        var header = CsvLine.Split(lines[headerIndex]);
        if (header is null) return Fail("Header row has an unterminated quote.");

        var columns = new Dictionary<string, int>(StringComparer.OrdinalIgnoreCase);
        for (var i = 0; i < header.Count; i++) columns.TryAdd(header[i].Trim(), i);
        var missing = Required.Where(r => !columns.ContainsKey(r)).ToList();
        if (missing.Count > 0) return Fail($"Missing required columns: {string.Join(", ", missing)}.");

        var errors = new List<string>();
        var subjects = new List<WatchlistSubject>();
        var references = new HashSet<string>(StringComparer.OrdinalIgnoreCase);
        var rows = 0;

        for (var i = headerIndex + 1; i < lines.Length; i++)
        {
            if (string.IsNullOrWhiteSpace(lines[i])) continue;
            if (++rows > MaxRows)
            {
                errors.Add($"More than {MaxRows} rows.");
                break;
            }

            var lineNumber = i + 1;
            var fields = CsvLine.Split(lines[i]);
            if (fields is null)
            {
                errors.Add($"Line {lineNumber}: unterminated quote.");
                continue;
            }

            string? Get(string name)
            {
                if (!columns.TryGetValue(name, out var idx) || idx >= fields.Count) return null;
                var value = fields[idx].Trim();
                return value.Length == 0 ? null : value;
            }

            var rowErrors = new List<string>();
            var reference = Get("reference");
            var name = Get("name");
            if (reference is null) rowErrors.Add("reference is required");
            else if (!references.Add(reference)) rowErrors.Add($"reference '{reference}' appears more than once");
            if (name is null) rowErrors.Add("name is required");
            if (!Enum.TryParse<WatchlistCategory>(Get("category"), true, out var category) || !Enum.IsDefined(category))
                rowErrors.Add("category must be PEP, SANCTIONS or WATCHLIST");
            if (!Enum.TryParse<SubjectKind>(Get("kind"), true, out var kind) || !Enum.IsDefined(kind))
                rowErrors.Add("kind must be INDIVIDUAL or ENTITY");

            var birthYears = new List<BirthYearRange>();
            if (Get("birthYear") is { } birthYear)
            {
                if (ParseYears(birthYear) is { } range) birthYears.Add(range);
                else rowErrors.Add("birthYear must be a year (1970) or a range (1968-1972)");
            }

            if (rowErrors.Count > 0)
            {
                errors.Add($"Line {lineNumber}: {string.Join("; ", rowErrors)}.");
                continue;
            }

            var names = new List<string> { name! };
            foreach (var alias in (Get("aliases") ?? "").Split(';', StringSplitOptions.RemoveEmptyEntries | StringSplitOptions.TrimEntries))
                if (!names.Contains(alias, StringComparer.OrdinalIgnoreCase)) names.Add(alias);
            var countries = Get("country") is { } country ? new[] { country } : Array.Empty<string>();

            subjects.Add(new WatchlistSubject(
                WatchlistSource.TENANT, reference!, category, kind, names, birthYears, countries, Get("notes")));
        }

        if (errors.Count == 0 && subjects.Count == 0) errors.Add("The file has no rows.");
        return errors.Count > 0
            ? new TenantWatchlistParseResult(errors, Array.Empty<WatchlistSubject>())
            : new TenantWatchlistParseResult(Array.Empty<string>(), subjects);
    }

    private static BirthYearRange? ParseYears(string text)
    {
        var parts = text.Split('-', StringSplitOptions.TrimEntries);
        if (parts.Length > 2) return null;
        var years = new List<int>();
        foreach (var part in parts)
        {
            if (!int.TryParse(part, NumberStyles.None, CultureInfo.InvariantCulture, out var year) || year is < 1900 or > 2100)
                return null;
            years.Add(year);
        }
        return new BirthYearRange(years.Min(), years.Max());
    }

    private static TenantWatchlistParseResult Fail(string error)
        => new(new[] { error }, Array.Empty<WatchlistSubject>());
}
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `dotnet test tests/Unit`
Expected: the new tests and the existing `TransactionCsvParser` tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/Aegis.Application tests/Unit/Screening/TenantWatchlistCsvParserTests.cs
git commit -m "feat(screening): tenant watchlist CSV parser; share CSV line splitting"
```

---

### Task 4: Storage — entities, customer date of birth, migration

**Files:**
- Create: `src/Aegis.Modules.Screening/Domain/WatchlistEntry.cs`, `WatchlistLoad.cs`, `TenantWatchlistEntry.cs`, `ScreeningMatch.cs`
- Create: `src/Aegis.Modules.Screening/Application/IScreeningRepositories.cs`
- Modify: `src/Aegis.Modules.Customers/Domain/Customer.cs` (optional `DateOfBirth`)
- Modify: `src/Aegis.Infrastructure/Persistence/Configurations/Risk/RiskConfigurations.cs` (rename `RiskJson` to `JsonColumn` and move it to `Configurations/JsonColumn.cs`)
- Create: `src/Aegis.Infrastructure/Persistence/Configurations/JsonColumn.cs`, `Configurations/Screening/ScreeningConfigurations.cs`
- Modify: `src/Aegis.Infrastructure/Persistence/Configurations/Customers/CustomerConfigurations.cs` (map `DateOfBirth`)
- Modify: `src/Aegis.Infrastructure/Persistence/AegisDbContext.cs` (DbSets, tenant filters)
- Create: `src/Aegis.Infrastructure/Persistence/Repositories/ScreeningRepositories.cs`
- Modify: `src/Aegis.Infrastructure/Aegis.Infrastructure.csproj` (reference Screening), `src/Aegis.Infrastructure/DependencyInjection.cs`
- Create: migration `AddScreening`
- Test: `tests/Unit/Screening/ScreeningMatchTests.cs`, `tests/Unit/Customers/CustomerDateOfBirthTests.cs`

**Interfaces:**
- Consumes: Task 1 types.
- Produces (namespace `Aegis.Modules.Screening.Domain` unless noted):
  - `WatchlistEntry` (global, not tenant-owned): `static From(WatchlistSubject, Guid loadId)`, `WatchlistSubject ToSubject()`
  - `WatchlistLoad` (global): `Id, Source, LoadedAt, PublishedAt, EntryCount, ContentHash`; `static Record(WatchlistSource, DateTimeOffset? publishedAt, int entryCount, string contentHash)`
  - `TenantWatchlistEntry : EntityBase`: `static From(TenantId, WatchlistSubject, string uploadedBy)`, `UploadedBy`, `WatchlistSubject ToSubject()`
  - `enum ScreeningMatchStatus { PENDING_REVIEW, CONFIRMED, FALSE_POSITIVE }`
  - `static class ScreeningTriggers { CustomerCreated, ListUpdated, Manual }` ("CUSTOMER_CREATED", "LIST_UPDATED", "MANUAL")
  - `ScreeningMatch : AggregateRoot`: `CustomerId, CustomerName, Source, ExternalId, Category, MatchedName, Score, BirthYearMatched, Subject, Trigger, Status, ReviewedBy, ReviewedAt, ReviewReason, AlertId`; `static Create(TenantId, ScreeningCandidate, PotentialMatch, string trigger)`; `Confirm(string reviewer, string reason, Guid alertId)`; `MarkFalsePositive(string reviewer, string reason)`
  - Ports (namespace `Aegis.Modules.Screening.Application`): `IWatchlistRepository`, `ITenantWatchlistRepository`, `IScreeningMatchRepository`, `record TenantWatchlistSummary(int EntryCount, DateTimeOffset? UploadedAt, string? UploadedBy)`, `record ScreeningMatchQuery(ScreeningMatchStatus? Status = null, Guid? CustomerId = null, int Page = 1, int PageSize = 50)` (signatures below)
  - `Customer.DateOfBirth` (`DateOnly?`); `Customer.CreateIndividual(..., DateOnly? dateOfBirth = null)`

- [ ] **Step 1: Write the failing domain tests**

Create `tests/Unit/Screening/ScreeningMatchTests.cs`:

```csharp
namespace Aegis.Tests.Unit.Screening;

using Aegis.Modules.Screening.Domain;
using Aegis.Shared.Domain;

public sealed class ScreeningMatchTests
{
    private static ScreeningMatch NewMatch()
    {
        var subject = new WatchlistSubject(
            WatchlistSource.UN, "QDi.901", WatchlistCategory.SANCTIONS, SubjectKind.INDIVIDUAL,
            new[] { "ZEPHANIAH OKOTH MBWANA" }, Array.Empty<BirthYearRange>(), Array.Empty<string>(), null);
        var candidate = new ScreeningCandidate(Guid.NewGuid(), "Zephaniah Mbwana", SubjectKind.INDIVIDUAL, null);
        return ScreeningMatch.Create(new TenantId(Guid.NewGuid()), candidate,
            new PotentialMatch(subject, "ZEPHANIAH OKOTH MBWANA", 0.9333, false), ScreeningTriggers.CustomerCreated);
    }

    [Fact]
    public void New_match_is_pending_and_copies_the_subject()
    {
        var match = NewMatch();
        Assert.Equal(ScreeningMatchStatus.PENDING_REVIEW, match.Status);
        Assert.Equal("QDi.901", match.ExternalId);
        Assert.Equal(WatchlistCategory.SANCTIONS, match.Category);
        Assert.Equal("Zephaniah Mbwana", match.CustomerName);
        Assert.Equal("ZEPHANIAH OKOTH MBWANA", match.Subject.Names[0]);
    }

    [Fact]
    public void Confirm_records_reviewer_reason_and_alert()
    {
        var match = NewMatch();
        var alertId = Guid.NewGuid();
        match.Confirm("user-1", "Same passport number", alertId);
        Assert.Equal(ScreeningMatchStatus.CONFIRMED, match.Status);
        Assert.Equal("user-1", match.ReviewedBy);
        Assert.Equal("Same passport number", match.ReviewReason);
        Assert.Equal(alertId, match.AlertId);
        Assert.NotNull(match.ReviewedAt);
    }

    [Fact]
    public void Review_requires_a_reason()
        => Assert.Throws<ArgumentException>(() => NewMatch().MarkFalsePositive("user-1", " "));

    [Fact]
    public void A_match_is_reviewed_once()
    {
        var match = NewMatch();
        match.MarkFalsePositive("user-1", "Different date of birth");
        Assert.Throws<InvalidOperationException>(() => match.Confirm("user-2", "Changed my mind", Guid.NewGuid()));
    }
}
```

Create `tests/Unit/Customers/CustomerDateOfBirthTests.cs`:

```csharp
namespace Aegis.Tests.Unit.Customers;

using Aegis.Modules.Customers.Domain;
using Aegis.Shared.Domain;

public sealed class CustomerDateOfBirthTests
{
    [Fact]
    public void Individual_keeps_an_optional_date_of_birth()
    {
        var customer = Customer.CreateIndividual(new TenantId(Guid.NewGuid()), null, "KE", "Jane", "Doe", new DateOnly(1980, 5, 1));
        Assert.Equal(new DateOnly(1980, 5, 1), customer.DateOfBirth);
        Assert.Null(Customer.CreateIndividual(new TenantId(Guid.NewGuid()), null, "KE", "Jane", "Doe").DateOfBirth);
    }
}
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `dotnet test tests/Unit --filter "FullyQualifiedName~ScreeningMatchTests|FullyQualifiedName~CustomerDateOfBirthTests"`
Expected: build fails.

- [ ] **Step 3: Implement the entities**

In `Customer.cs` add the property and parameter:

```csharp
public DateOnly? DateOfBirth { get; private set; }

public static Customer CreateIndividual(
    TenantId tenantId,
    string? externalReference,
    string country,
    string firstName,
    string lastName,
    DateOnly? dateOfBirth = null)
```

and set `DateOfBirth = dateOfBirth,` in the initializer.

Create `src/Aegis.Modules.Screening/Domain/WatchlistEntry.cs`:

```csharp
namespace Aegis.Modules.Screening.Domain;

/// <summary>A UN or OFAC subject. Shared by every tenant, so it is not tenant-owned.</summary>
public sealed class WatchlistEntry
{
    public Guid Id { get; private set; }
    public WatchlistSource Source { get; private set; }
    public string ExternalId { get; private set; } = null!;
    public WatchlistCategory Category { get; private set; }
    public SubjectKind Kind { get; private set; }
    public List<string> Names { get; private set; } = new();
    public List<BirthYearRange> BirthYears { get; private set; } = new();
    public List<string> Countries { get; private set; } = new();
    public string? Detail { get; private set; }
    public Guid LoadId { get; private set; }

    private WatchlistEntry() { }

    public static WatchlistEntry From(WatchlistSubject s, Guid loadId) => new()
    {
        Id = Guid.NewGuid(),
        Source = s.Source,
        ExternalId = s.ExternalId,
        Category = s.Category,
        Kind = s.Kind,
        Names = s.Names.ToList(),
        BirthYears = s.BirthYears.ToList(),
        Countries = s.Countries.ToList(),
        Detail = s.Detail,
        LoadId = loadId
    };

    public WatchlistSubject ToSubject() => new(Source, ExternalId, Category, Kind, Names, BirthYears, Countries, Detail);
}
```

Create `src/Aegis.Modules.Screening/Domain/WatchlistLoad.cs`:

```csharp
namespace Aegis.Modules.Screening.Domain;

/// <summary>One successful load of a global list; the newest row per source is the current list.</summary>
public sealed class WatchlistLoad
{
    public Guid Id { get; private set; }
    public WatchlistSource Source { get; private set; }
    public DateTimeOffset LoadedAt { get; private set; }
    public DateTimeOffset? PublishedAt { get; private set; }
    public int EntryCount { get; private set; }
    public string ContentHash { get; private set; } = null!;

    private WatchlistLoad() { }

    public static WatchlistLoad Record(WatchlistSource source, DateTimeOffset? publishedAt, int entryCount, string contentHash) => new()
    {
        Id = Guid.NewGuid(),
        Source = source,
        LoadedAt = DateTimeOffset.UtcNow,
        PublishedAt = publishedAt,
        EntryCount = entryCount,
        ContentHash = contentHash
    };
}
```

Create `src/Aegis.Modules.Screening/Domain/TenantWatchlistEntry.cs`:

```csharp
namespace Aegis.Modules.Screening.Domain;

using Aegis.Shared.Domain;

public sealed class TenantWatchlistEntry : EntityBase
{
    public string ExternalId { get; private set; } = null!;
    public WatchlistCategory Category { get; private set; }
    public SubjectKind Kind { get; private set; }
    public List<string> Names { get; private set; } = new();
    public List<BirthYearRange> BirthYears { get; private set; } = new();
    public List<string> Countries { get; private set; } = new();
    public string? Detail { get; private set; }
    public string UploadedBy { get; private set; } = null!;

    private TenantWatchlistEntry() { }

    public static TenantWatchlistEntry From(TenantId tenantId, WatchlistSubject s, string uploadedBy)
    {
        var now = DateTimeOffset.UtcNow;
        return new TenantWatchlistEntry
        {
            Id = Guid.NewGuid(),
            TenantId = tenantId,
            ExternalId = s.ExternalId,
            Category = s.Category,
            Kind = s.Kind,
            Names = s.Names.ToList(),
            BirthYears = s.BirthYears.ToList(),
            Countries = s.Countries.ToList(),
            Detail = s.Detail,
            UploadedBy = uploadedBy,
            CreatedAt = now,
            UpdatedAt = now
        };
    }

    public WatchlistSubject ToSubject()
        => new(WatchlistSource.TENANT, ExternalId, Category, Kind, Names, BirthYears, Countries, Detail);
}
```

Create `src/Aegis.Modules.Screening/Domain/ScreeningMatch.cs`:

```csharp
namespace Aegis.Modules.Screening.Domain;

using Aegis.Shared.Domain;

public enum ScreeningMatchStatus { PENDING_REVIEW, CONFIRMED, FALSE_POSITIVE }

public static class ScreeningTriggers
{
    public const string CustomerCreated = "CUSTOMER_CREATED";
    public const string ListUpdated = "LIST_UPDATED";
    public const string Manual = "MANUAL";
}

/// <summary>A possible match awaiting or after human review. One per customer and listed subject.</summary>
public sealed class ScreeningMatch : AggregateRoot
{
    public Guid CustomerId { get; private set; }
    public string CustomerName { get; private set; } = null!;
    public WatchlistSource Source { get; private set; }
    public string ExternalId { get; private set; } = null!;
    public WatchlistCategory Category { get; private set; }
    public string MatchedName { get; private set; } = null!;
    public double Score { get; private set; }
    public bool BirthYearMatched { get; private set; }

    /// <summary>The listed subject as it was when matched; lists change, the review record must not.</summary>
    public WatchlistSubject Subject { get; private set; } = null!;

    public string Trigger { get; private set; } = null!;
    public ScreeningMatchStatus Status { get; private set; }
    public string? ReviewedBy { get; private set; }
    public DateTimeOffset? ReviewedAt { get; private set; }
    public string? ReviewReason { get; private set; }
    public Guid? AlertId { get; private set; }

    private ScreeningMatch() { }

    public static ScreeningMatch Create(TenantId tenantId, ScreeningCandidate candidate, PotentialMatch match, string trigger)
    {
        var now = DateTimeOffset.UtcNow;
        return new ScreeningMatch
        {
            Id = Guid.NewGuid(),
            TenantId = tenantId,
            CustomerId = candidate.CustomerId,
            CustomerName = candidate.Name,
            Source = match.Subject.Source,
            ExternalId = match.Subject.ExternalId,
            Category = match.Subject.Category,
            MatchedName = match.MatchedName,
            Score = match.Score,
            BirthYearMatched = match.BirthYearMatched,
            Subject = match.Subject,
            Trigger = trigger,
            Status = ScreeningMatchStatus.PENDING_REVIEW,
            CreatedAt = now,
            UpdatedAt = now
        };
    }

    public void Confirm(string reviewer, string reason, Guid alertId)
    {
        Review(ScreeningMatchStatus.CONFIRMED, reviewer, reason);
        AlertId = alertId;
    }

    public void MarkFalsePositive(string reviewer, string reason) => Review(ScreeningMatchStatus.FALSE_POSITIVE, reviewer, reason);

    private void Review(ScreeningMatchStatus status, string reviewer, string reason)
    {
        if (string.IsNullOrWhiteSpace(reason)) throw new ArgumentException("A review reason is required.", nameof(reason));
        if (Status != ScreeningMatchStatus.PENDING_REVIEW) throw new InvalidOperationException($"Match is already {Status}.");
        Status = status;
        ReviewedBy = reviewer;
        ReviewReason = reason.Trim();
        ReviewedAt = DateTimeOffset.UtcNow;
        UpdatedAt = ReviewedAt.Value;
    }
}
```

Run the unit tests from Step 2. Expected: PASS.

- [ ] **Step 4: Repository ports**

Create `src/Aegis.Modules.Screening/Application/IScreeningRepositories.cs`:

```csharp
namespace Aegis.Modules.Screening.Application;

using Aegis.Modules.Screening.Domain;
using Aegis.Shared.Contracts;
using Aegis.Shared.Domain;

public interface IWatchlistRepository
{
    /// <summary>Newest load per source.</summary>
    Task<IReadOnlyList<WatchlistLoad>> ListLatestLoadsAsync(CancellationToken cancellationToken = default);

    Task<WatchlistLoad?> GetLatestLoadAsync(WatchlistSource source, CancellationToken cancellationToken = default);

    /// <summary>Time of the newest load of any source; changes whenever a list is replaced.</summary>
    Task<DateTimeOffset?> GetLastLoadedAtAsync(CancellationToken cancellationToken = default);

    Task<IReadOnlyList<WatchlistSubject>> ListSubjectsAsync(CancellationToken cancellationToken = default);

    /// <summary>Replaces every entry of the load's source and records the load, in one transaction. Saves immediately.</summary>
    Task ReplaceSourceAsync(WatchlistLoad load, IReadOnlyList<WatchlistSubject> subjects, CancellationToken cancellationToken = default);
}

public sealed record TenantWatchlistSummary(int EntryCount, DateTimeOffset? UploadedAt, string? UploadedBy);

public interface ITenantWatchlistRepository
{
    Task<IReadOnlyList<WatchlistSubject>> ListSubjectsAsync(TenantId tenantId, CancellationToken cancellationToken = default);

    Task<TenantWatchlistSummary> GetSummaryAsync(TenantId tenantId, CancellationToken cancellationToken = default);

    /// <summary>Removes the tenant's entries and adds <paramref name="entries"/>; the caller saves.</summary>
    Task ReplaceAsync(TenantId tenantId, IReadOnlyList<TenantWatchlistEntry> entries, CancellationToken cancellationToken = default);
}

public sealed record ScreeningMatchQuery(ScreeningMatchStatus? Status = null, Guid? CustomerId = null, int Page = 1, int PageSize = 50);

public interface IScreeningMatchRepository
{
    Task<ScreeningMatch?> GetAsync(TenantId tenantId, Guid id, CancellationToken cancellationToken = default);

    /// <summary>Source and external id of every match already recorded for the customer, in any status.</summary>
    Task<IReadOnlySet<(WatchlistSource Source, string ExternalId)>> ListKeysAsync(
        TenantId tenantId, Guid customerId, CancellationToken cancellationToken = default);

    /// <summary>Newest first.</summary>
    Task<PagedResult<ScreeningMatch>> ListAsync(TenantId tenantId, ScreeningMatchQuery query, CancellationToken cancellationToken = default);

    Task<int> CountPendingAsync(TenantId tenantId, CancellationToken cancellationToken = default);

    Task AddAsync(ScreeningMatch match, CancellationToken cancellationToken = default);
}
```

- [ ] **Step 5: EF mapping**

Move `RiskJson` out of `RiskConfigurations.cs` into `src/Aegis.Infrastructure/Persistence/Configurations/JsonColumn.cs` as `internal static class JsonColumn` (namespace `Aegis.Infrastructure.Persistence.Configurations`, body unchanged), and replace `RiskJson.` with `JsonColumn.` in `RiskConfigurations.cs` (add `using Aegis.Infrastructure.Persistence.Configurations;`).

In `CustomerConfigurations.cs`, inside the `Customer` configuration, add:

```csharp
builder.Property(x => x.DateOfBirth);
```

Create `src/Aegis.Infrastructure/Persistence/Configurations/Screening/ScreeningConfigurations.cs`:

```csharp
namespace Aegis.Infrastructure.Persistence.Configurations.Screening;

using Aegis.Modules.Screening.Domain;
using Aegis.Shared.Domain;
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;

internal static class ScreeningMapping
{
    public static void Lists<T>(EntityTypeBuilder<T> builder) where T : class
    {
        builder.Property<List<string>>("Names").HasColumnName("names_json").HasColumnType("jsonb")
            .HasConversion(v => JsonColumn.Write(v), v => JsonColumn.Read<List<string>>(v))
            .Metadata.SetValueComparer(JsonColumn.Comparer<List<string>>());
        builder.Property<List<BirthYearRange>>("BirthYears").HasColumnName("birth_years_json").HasColumnType("jsonb")
            .HasConversion(v => JsonColumn.Write(v), v => JsonColumn.Read<List<BirthYearRange>>(v))
            .Metadata.SetValueComparer(JsonColumn.Comparer<List<BirthYearRange>>());
        builder.Property<List<string>>("Countries").HasColumnName("countries_json").HasColumnType("jsonb")
            .HasConversion(v => JsonColumn.Write(v), v => JsonColumn.Read<List<string>>(v))
            .Metadata.SetValueComparer(JsonColumn.Comparer<List<string>>());
    }
}

public sealed class WatchlistEntryConfiguration : IEntityTypeConfiguration<WatchlistEntry>
{
    public void Configure(EntityTypeBuilder<WatchlistEntry> builder)
    {
        builder.ToTable("watchlist_entries", "screening");
        builder.HasKey(x => x.Id);
        builder.Property(x => x.Source).HasConversion<string>().HasMaxLength(16);
        builder.Property(x => x.ExternalId).HasMaxLength(64).IsRequired();
        builder.Property(x => x.Category).HasConversion<string>().HasMaxLength(16);
        builder.Property(x => x.Kind).HasConversion<string>().HasMaxLength(16);
        builder.Property(x => x.Detail).HasMaxLength(500);
        builder.Property(x => x.LoadId).HasColumnName("load_id");
        ScreeningMapping.Lists(builder);
        builder.HasIndex(x => new { x.Source, x.ExternalId }).IsUnique();
    }
}

public sealed class WatchlistLoadConfiguration : IEntityTypeConfiguration<WatchlistLoad>
{
    public void Configure(EntityTypeBuilder<WatchlistLoad> builder)
    {
        builder.ToTable("watchlist_loads", "screening");
        builder.HasKey(x => x.Id);
        builder.Property(x => x.Source).HasConversion<string>().HasMaxLength(16);
        builder.Property(x => x.ContentHash).HasMaxLength(64).IsRequired();
        builder.HasIndex(x => new { x.Source, x.LoadedAt });
    }
}

public sealed class TenantWatchlistEntryConfiguration : IEntityTypeConfiguration<TenantWatchlistEntry>
{
    public void Configure(EntityTypeBuilder<TenantWatchlistEntry> builder)
    {
        builder.ToTable("tenant_watchlist_entries", "screening");
        builder.HasKey(x => x.Id);
        builder.Property(x => x.TenantId)
            .HasConversion(id => id.Value, value => new TenantId(value))
            .HasColumnName("tenant_id")
            .IsRequired();
        builder.Property(x => x.ExternalId).HasMaxLength(64).IsRequired();
        builder.Property(x => x.Category).HasConversion<string>().HasMaxLength(16);
        builder.Property(x => x.Kind).HasConversion<string>().HasMaxLength(16);
        builder.Property(x => x.Detail).HasMaxLength(500);
        builder.Property(x => x.UploadedBy).HasMaxLength(200).IsRequired();
        ScreeningMapping.Lists(builder);
        builder.HasIndex(x => new { x.TenantId, x.ExternalId }).IsUnique();
    }
}

public sealed class ScreeningMatchConfiguration : IEntityTypeConfiguration<ScreeningMatch>
{
    public void Configure(EntityTypeBuilder<ScreeningMatch> builder)
    {
        builder.ToTable("screening_matches", "screening");
        builder.HasKey(x => x.Id);
        builder.Property(x => x.TenantId)
            .HasConversion(id => id.Value, value => new TenantId(value))
            .HasColumnName("tenant_id")
            .IsRequired();
        builder.Property(x => x.CustomerId).HasColumnName("customer_id").IsRequired();
        builder.Property(x => x.CustomerName).HasMaxLength(300).IsRequired();
        builder.Property(x => x.Source).HasConversion<string>().HasMaxLength(16);
        builder.Property(x => x.ExternalId).HasMaxLength(64).IsRequired();
        builder.Property(x => x.Category).HasConversion<string>().HasMaxLength(16);
        builder.Property(x => x.MatchedName).HasMaxLength(300).IsRequired();
        builder.Property(x => x.Trigger).HasMaxLength(32).IsRequired();
        builder.Property(x => x.Status).HasConversion<string>().HasMaxLength(16);
        builder.Property(x => x.ReviewedBy).HasMaxLength(200);
        builder.Property(x => x.ReviewReason).HasMaxLength(2000);
        builder.Property(x => x.AlertId).HasColumnName("alert_id");
        builder.Property(x => x.Subject).HasColumnName("subject_json").HasColumnType("jsonb")
            .HasConversion(v => JsonColumn.Write(v), v => JsonColumn.Read<WatchlistSubjectJson>(v).ToSubject())
            .Metadata.SetValueComparer(new Microsoft.EntityFrameworkCore.ChangeTracking.ValueComparer<WatchlistSubject>(
                (a, b) => JsonColumn.Write(a) == JsonColumn.Write(b),
                v => JsonColumn.Write(v).GetHashCode(),
                v => v));
        builder.HasIndex(x => new { x.TenantId, x.CustomerId, x.Source, x.ExternalId }).IsUnique();
        builder.HasIndex(x => new { x.TenantId, x.Status, x.CreatedAt });
        builder.Ignore(x => x.DomainEvents);
    }

    /// <summary>
    /// <see cref="JsonColumn.Read{T}"/> needs a parameterless type; this mirrors <see cref="WatchlistSubject"/>.
    /// </summary>
    private sealed class WatchlistSubjectJson
    {
        public WatchlistSource Source { get; set; }
        public string ExternalId { get; set; } = "";
        public WatchlistCategory Category { get; set; }
        public SubjectKind Kind { get; set; }
        public List<string> Names { get; set; } = new();
        public List<BirthYearRange> BirthYears { get; set; } = new();
        public List<string> Countries { get; set; } = new();
        public string? Detail { get; set; }

        public WatchlistSubject ToSubject() => new(Source, ExternalId, Category, Kind, Names, BirthYears, Countries, Detail);
    }
}
```

`BirthYearRange` is a positional record without a parameterless constructor; `System.Text.Json` binds it through its constructor, so `JsonColumn.Read<List<BirthYearRange>>` works. `Read<T>` only needs `new()` on the list type.

In `AegisDbContext.cs` add:

```csharp
public DbSet<WatchlistEntry> WatchlistEntries => Set<WatchlistEntry>();
public DbSet<WatchlistLoad> WatchlistLoads => Set<WatchlistLoad>();
public DbSet<TenantWatchlistEntry> TenantWatchlistEntries => Set<TenantWatchlistEntry>();
public DbSet<ScreeningMatch> ScreeningMatches => Set<ScreeningMatch>();
```

and in `OnModelCreating`, next to the other filters:

```csharp
modelBuilder.Entity<TenantWatchlistEntry>().HasQueryFilter(e => !TenantFilterEnabled || e.TenantId == CurrentTenantId);
modelBuilder.Entity<ScreeningMatch>().HasQueryFilter(e => !TenantFilterEnabled || e.TenantId == CurrentTenantId);
```

Add `<ProjectReference Include="..\Aegis.Modules.Screening\Aegis.Modules.Screening.csproj" />` to `Aegis.Infrastructure.csproj`.

- [ ] **Step 6: Repositories**

Create `src/Aegis.Infrastructure/Persistence/Repositories/ScreeningRepositories.cs`:

```csharp
namespace Aegis.Infrastructure.Persistence.Repositories;

using Aegis.Modules.Screening.Application;
using Aegis.Modules.Screening.Domain;
using Aegis.Shared.Contracts;
using Aegis.Shared.Domain;
using Microsoft.EntityFrameworkCore;

public sealed class WatchlistRepository : IWatchlistRepository
{
    private readonly AegisDbContext _db;

    public WatchlistRepository(AegisDbContext db) => _db = db;

    public async Task<IReadOnlyList<WatchlistLoad>> ListLatestLoadsAsync(CancellationToken cancellationToken = default)
    {
        var loads = await _db.WatchlistLoads.AsNoTracking().ToListAsync(cancellationToken);
        return loads.GroupBy(l => l.Source).Select(g => g.MaxBy(l => l.LoadedAt)!).OrderBy(l => l.Source).ToList();
    }

    public Task<WatchlistLoad?> GetLatestLoadAsync(WatchlistSource source, CancellationToken cancellationToken = default)
        => _db.WatchlistLoads.AsNoTracking()
            .Where(l => l.Source == source)
            .OrderByDescending(l => l.LoadedAt)
            .FirstOrDefaultAsync(cancellationToken);

    public async Task<DateTimeOffset?> GetLastLoadedAtAsync(CancellationToken cancellationToken = default)
        => await _db.WatchlistLoads.AsNoTracking().MaxAsync(l => (DateTimeOffset?)l.LoadedAt, cancellationToken);

    public async Task<IReadOnlyList<WatchlistSubject>> ListSubjectsAsync(CancellationToken cancellationToken = default)
    {
        var entries = await _db.WatchlistEntries.AsNoTracking().ToListAsync(cancellationToken);
        return entries.Select(e => e.ToSubject()).ToList();
    }

    public async Task ReplaceSourceAsync(WatchlistLoad load, IReadOnlyList<WatchlistSubject> subjects, CancellationToken cancellationToken = default)
    {
        await using var transaction = await _db.Database.BeginTransactionAsync(cancellationToken);
        await _db.WatchlistEntries.Where(e => e.Source == load.Source).ExecuteDeleteAsync(cancellationToken);
        await _db.WatchlistLoads.AddAsync(load, cancellationToken);
        await _db.WatchlistEntries.AddRangeAsync(subjects.Select(s => WatchlistEntry.From(s, load.Id)), cancellationToken);
        await _db.SaveChangesAsync(cancellationToken);
        await transaction.CommitAsync(cancellationToken);
        _db.ChangeTracker.Clear();
    }
}

public sealed class TenantWatchlistRepository : ITenantWatchlistRepository
{
    private readonly AegisDbContext _db;

    public TenantWatchlistRepository(AegisDbContext db) => _db = db;

    public async Task<IReadOnlyList<WatchlistSubject>> ListSubjectsAsync(TenantId tenantId, CancellationToken cancellationToken = default)
    {
        var entries = await _db.TenantWatchlistEntries.AsNoTracking()
            .Where(e => e.TenantId == tenantId)
            .ToListAsync(cancellationToken);
        return entries.Select(e => e.ToSubject()).ToList();
    }

    public async Task<TenantWatchlistSummary> GetSummaryAsync(TenantId tenantId, CancellationToken cancellationToken = default)
    {
        var rows = _db.TenantWatchlistEntries.AsNoTracking().Where(e => e.TenantId == tenantId);
        var count = await rows.CountAsync(cancellationToken);
        var latest = await rows.OrderByDescending(e => e.CreatedAt)
            .Select(e => new { e.CreatedAt, e.UploadedBy })
            .FirstOrDefaultAsync(cancellationToken);
        return new TenantWatchlistSummary(count, latest?.CreatedAt, latest?.UploadedBy);
    }

    public async Task ReplaceAsync(TenantId tenantId, IReadOnlyList<TenantWatchlistEntry> entries, CancellationToken cancellationToken = default)
    {
        var existing = await _db.TenantWatchlistEntries.Where(e => e.TenantId == tenantId).ToListAsync(cancellationToken);
        _db.TenantWatchlistEntries.RemoveRange(existing);
        await _db.TenantWatchlistEntries.AddRangeAsync(entries, cancellationToken);
    }
}

public sealed class ScreeningMatchRepository : IScreeningMatchRepository
{
    private readonly AegisDbContext _db;

    public ScreeningMatchRepository(AegisDbContext db) => _db = db;

    public Task<ScreeningMatch?> GetAsync(TenantId tenantId, Guid id, CancellationToken cancellationToken = default)
        => _db.ScreeningMatches.FirstOrDefaultAsync(m => m.TenantId == tenantId && m.Id == id, cancellationToken);

    public async Task<IReadOnlySet<(WatchlistSource Source, string ExternalId)>> ListKeysAsync(
        TenantId tenantId, Guid customerId, CancellationToken cancellationToken = default)
    {
        var keys = await _db.ScreeningMatches.AsNoTracking()
            .Where(m => m.TenantId == tenantId && m.CustomerId == customerId)
            .Select(m => new { m.Source, m.ExternalId })
            .ToListAsync(cancellationToken);
        var local = _db.ScreeningMatches.Local
            .Where(m => m.TenantId == tenantId && m.CustomerId == customerId)
            .Select(m => new { m.Source, m.ExternalId });
        return keys.Concat(local).Select(k => (k.Source, k.ExternalId)).ToHashSet();
    }

    public async Task<PagedResult<ScreeningMatch>> ListAsync(TenantId tenantId, ScreeningMatchQuery query, CancellationToken cancellationToken = default)
    {
        var page = Math.Max(1, query.Page);
        var pageSize = Math.Clamp(query.PageSize, 1, 200);
        var rows = _db.ScreeningMatches.AsNoTracking().Where(m => m.TenantId == tenantId);
        if (query.Status is { } status) rows = rows.Where(m => m.Status == status);
        if (query.CustomerId is { } customerId) rows = rows.Where(m => m.CustomerId == customerId);

        var total = await rows.LongCountAsync(cancellationToken);
        var items = await rows.OrderByDescending(m => m.CreatedAt).ThenBy(m => m.Id)
            .Skip((page - 1) * pageSize).Take(pageSize)
            .ToListAsync(cancellationToken);
        return new PagedResult<ScreeningMatch> { Items = items, Page = page, PageSize = pageSize, TotalCount = total };
    }

    public Task<int> CountPendingAsync(TenantId tenantId, CancellationToken cancellationToken = default)
        => _db.ScreeningMatches.AsNoTracking()
            .CountAsync(m => m.TenantId == tenantId && m.Status == ScreeningMatchStatus.PENDING_REVIEW, cancellationToken);

    public async Task AddAsync(ScreeningMatch match, CancellationToken cancellationToken = default)
        => await _db.ScreeningMatches.AddAsync(match, cancellationToken);
}
```

`ListKeysAsync` includes unsaved matches from the change tracker so a tenant-wide screen that saves per batch never adds the same subject twice for one customer.

Register in `DependencyInjection.cs` after the risk registrations:

```csharp
services.AddScoped<IWatchlistRepository, WatchlistRepository>();
services.AddScoped<ITenantWatchlistRepository, TenantWatchlistRepository>();
services.AddScoped<IScreeningMatchRepository, ScreeningMatchRepository>();
```

with `using Aegis.Modules.Screening.Application;`.

- [ ] **Step 7: Migration**

```bash
dotnet ef migrations add AddScreening -p src/Aegis.Infrastructure -s src/Aegis.Api
```

Open the generated migration and check it creates schema `screening` with the four tables and indexes above, adds nullable `DateOfBirth` (`date`) to `customers.customers`, and changes nothing else. If it touches other tables, stop and fix the mapping first.

- [ ] **Step 8: Build and run all tests**

Run: `dotnet build && dotnet test tests/Unit && dotnet test tests/Integration`
Expected: all pass (integration applies the migration to `aegis_test`).

- [ ] **Step 9: Commit**

```bash
git add src tests/Unit
git commit -m "feat(screening): watchlist, tenant list and match storage; customer date of birth"
```

---

### Task 5: List refresh — download, verify, replace, daily job

**Files:**
- Create: `src/Aegis.Application/Screening/WatchlistRefresher.cs`, `src/Aegis.Application/Screening/IGlobalWatchlist.cs`
- Create: `src/Aegis.Infrastructure/Screening/WatchlistDownloader.cs`, `GlobalWatchlistCache.cs`
- Modify: `src/Aegis.Infrastructure/DependencyInjection.cs`
- Modify: `tests/Integration/Identity/AuthAndAuditTests.cs` (`AegisApiFactory` settings)
- Test: `tests/Unit/Screening/WatchlistRefresherTests.cs`, `tests/Integration/Screening/WatchlistRefreshTests.cs`

The daily background job is added in Task 6, because it rescreens customers.

**Interfaces:**
- Consumes: parsers (Task 2), `IWatchlistRepository` (Task 4).
- Produces (namespace `Aegis.Application.Screening`):
  - `enum WatchlistRefreshStatus { LOADED, UNCHANGED, FAILED }`
  - `record WatchlistRefreshResult(WatchlistSource Source, WatchlistRefreshStatus Status, int EntryCount, string? Error)`
  - `interface IWatchlistDownloader { Task<byte[]> DownloadAsync(WatchlistSource source, CancellationToken cancellationToken); }`
  - `interface IWatchlistRefresher { IReadOnlyList<WatchlistSource> Sources { get; } Task<WatchlistRefreshResult> RefreshAsync(WatchlistSource source, CancellationToken cancellationToken = default); }`
  - `interface IGlobalWatchlist { Task<WatchlistIndex> GetAsync(CancellationToken cancellationToken = default); }`
- Config keys: `Screening:Sources:UN:Url`, `Screening:Sources:UN:Path`, `Screening:Sources:OFAC:Url`, `Screening:Sources:OFAC:Path` (a path wins over a URL).

- [ ] **Step 1: Write the failing unit tests**

Create `tests/Unit/Screening/WatchlistRefresherTests.cs`:

```csharp
namespace Aegis.Tests.Unit.Screening;

using Aegis.Application.Screening;
using Aegis.Modules.Screening.Application;
using Aegis.Modules.Screening.Domain;
using Microsoft.Extensions.Logging.Abstractions;

public sealed class WatchlistRefresherTests
{
    private static byte[] Fixture(string name)
        => File.ReadAllBytes(Path.Combine(AppContext.BaseDirectory, "Fixtures", "screening", name));

    [Fact]
    public async Task First_load_replaces_the_source()
    {
        var repo = new FakeWatchlistRepository();
        var refresher = new WatchlistRefresher(new FakeDownloader(Fixture("un-sample.xml")), repo, NullLogger<WatchlistRefresher>.Instance);

        var result = await refresher.RefreshAsync(WatchlistSource.UN);

        Assert.Equal(WatchlistRefreshStatus.LOADED, result.Status);
        Assert.Equal(3, result.EntryCount);
        Assert.Equal(3, repo.Subjects.Count);
        Assert.Equal(3, Assert.Single(repo.Loads).EntryCount);
    }

    [Fact]
    public async Task Same_file_again_is_unchanged()
    {
        var repo = new FakeWatchlistRepository();
        var refresher = new WatchlistRefresher(new FakeDownloader(Fixture("un-sample.xml")), repo, NullLogger<WatchlistRefresher>.Instance);
        await refresher.RefreshAsync(WatchlistSource.UN);

        var result = await refresher.RefreshAsync(WatchlistSource.UN);

        Assert.Equal(WatchlistRefreshStatus.UNCHANGED, result.Status);
        Assert.Single(repo.Loads);
    }

    [Fact]
    public async Task A_file_with_far_fewer_entries_is_rejected_as_truncated()
    {
        var repo = new FakeWatchlistRepository();
        repo.Loads.Add(WatchlistLoad.Record(WatchlistSource.UN, null, 1000, "previous"));
        var refresher = new WatchlistRefresher(new FakeDownloader(Fixture("un-sample.xml")), repo, NullLogger<WatchlistRefresher>.Instance);

        var result = await refresher.RefreshAsync(WatchlistSource.UN);

        Assert.Equal(WatchlistRefreshStatus.FAILED, result.Status);
        Assert.Contains("3 entries", result.Error);
        Assert.Empty(repo.Subjects);
    }

    [Fact]
    public async Task Download_or_parse_failure_is_reported_not_thrown()
    {
        var refresher = new WatchlistRefresher(new FakeDownloader("<html/>"u8.ToArray()), new FakeWatchlistRepository(), NullLogger<WatchlistRefresher>.Instance);

        var result = await refresher.RefreshAsync(WatchlistSource.OFAC);

        Assert.Equal(WatchlistRefreshStatus.FAILED, result.Status);
        Assert.Equal("Not an OFAC SDN list.", result.Error);
    }

    private sealed class FakeDownloader(byte[] bytes) : IWatchlistDownloader
    {
        public Task<byte[]> DownloadAsync(WatchlistSource source, CancellationToken cancellationToken) => Task.FromResult(bytes);
    }

    private sealed class FakeWatchlistRepository : IWatchlistRepository
    {
        public List<WatchlistLoad> Loads { get; } = new();
        public List<WatchlistSubject> Subjects { get; } = new();

        public Task<IReadOnlyList<WatchlistLoad>> ListLatestLoadsAsync(CancellationToken cancellationToken = default)
            => Task.FromResult<IReadOnlyList<WatchlistLoad>>(Loads);

        public Task<WatchlistLoad?> GetLatestLoadAsync(WatchlistSource source, CancellationToken cancellationToken = default)
            => Task.FromResult(Loads.Where(l => l.Source == source).MaxBy(l => l.LoadedAt));

        public Task<DateTimeOffset?> GetLastLoadedAtAsync(CancellationToken cancellationToken = default)
            => Task.FromResult(Loads.Count == 0 ? (DateTimeOffset?)null : Loads.Max(l => l.LoadedAt));

        public Task<IReadOnlyList<WatchlistSubject>> ListSubjectsAsync(CancellationToken cancellationToken = default)
            => Task.FromResult<IReadOnlyList<WatchlistSubject>>(Subjects);

        public Task ReplaceSourceAsync(WatchlistLoad load, IReadOnlyList<WatchlistSubject> subjects, CancellationToken cancellationToken = default)
        {
            Subjects.RemoveAll(s => s.Source == load.Source);
            Subjects.AddRange(subjects);
            Loads.Add(load);
            return Task.CompletedTask;
        }
    }
}
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `dotnet test tests/Unit --filter "FullyQualifiedName~WatchlistRefresherTests"`
Expected: build fails.

- [ ] **Step 3: Implement the refresher and global list port**

Create `src/Aegis.Application/Screening/IGlobalWatchlist.cs`:

```csharp
namespace Aegis.Application.Screening;

using Aegis.Modules.Screening.Domain;

/// <summary>The current UN and OFAC subjects, prepared for screening and reloaded when a list changes.</summary>
public interface IGlobalWatchlist
{
    Task<WatchlistIndex> GetAsync(CancellationToken cancellationToken = default);
}
```

Create `src/Aegis.Application/Screening/WatchlistRefresher.cs`:

```csharp
namespace Aegis.Application.Screening;

using System.Security.Cryptography;
using Aegis.Modules.Screening.Application;
using Aegis.Modules.Screening.Domain;
using Microsoft.Extensions.Logging;

public enum WatchlistRefreshStatus { LOADED, UNCHANGED, FAILED }

public sealed record WatchlistRefreshResult(WatchlistSource Source, WatchlistRefreshStatus Status, int EntryCount, string? Error);

public interface IWatchlistDownloader
{
    Task<byte[]> DownloadAsync(WatchlistSource source, CancellationToken cancellationToken);
}

public interface IWatchlistRefresher
{
    IReadOnlyList<WatchlistSource> Sources { get; }

    Task<WatchlistRefreshResult> RefreshAsync(WatchlistSource source, CancellationToken cancellationToken = default);
}

public sealed class WatchlistRefresher : IWatchlistRefresher
{
    /// <summary>A new file with less than this share of the previous entry count is treated as truncated.</summary>
    public const double MinimumRetainedShare = 0.5;

    private readonly IWatchlistDownloader _downloader;
    private readonly IWatchlistRepository _lists;
    private readonly ILogger<WatchlistRefresher> _logger;

    public WatchlistRefresher(IWatchlistDownloader downloader, IWatchlistRepository lists, ILogger<WatchlistRefresher> logger)
    {
        _downloader = downloader;
        _lists = lists;
        _logger = logger;
    }

    public IReadOnlyList<WatchlistSource> Sources { get; } = new[] { WatchlistSource.UN, WatchlistSource.OFAC };

    public async Task<WatchlistRefreshResult> RefreshAsync(WatchlistSource source, CancellationToken cancellationToken = default)
    {
        try
        {
            var bytes = await _downloader.DownloadAsync(source, cancellationToken);
            var hash = Convert.ToHexString(SHA256.HashData(bytes));
            var latest = await _lists.GetLatestLoadAsync(source, cancellationToken);
            if (latest?.ContentHash == hash)
                return new WatchlistRefreshResult(source, WatchlistRefreshStatus.UNCHANGED, latest.EntryCount, null);

            using var stream = new MemoryStream(bytes);
            var parsed = source switch
            {
                WatchlistSource.UN => UnSanctionsParser.Parse(stream),
                WatchlistSource.OFAC => OfacSdnParser.Parse(stream),
                _ => throw new ArgumentOutOfRangeException(nameof(source), source, "Only UN and OFAC are downloaded.")
            };

            var previous = latest?.EntryCount ?? 0;
            if (parsed.Subjects.Count == 0 || parsed.Subjects.Count < previous * MinimumRetainedShare)
                return Failed(source, $"Rejected: {parsed.Subjects.Count} entries against {previous} in the current list.");

            await _lists.ReplaceSourceAsync(
                WatchlistLoad.Record(source, parsed.PublishedAt, parsed.Subjects.Count, hash), parsed.Subjects, cancellationToken);
            _logger.LogInformation("Loaded {Count} {Source} watchlist entries", parsed.Subjects.Count, source);
            return new WatchlistRefreshResult(source, WatchlistRefreshStatus.LOADED, parsed.Subjects.Count, null);
        }
        catch (Exception ex) when (ex is not OperationCanceledException)
        {
            _logger.LogError(ex, "Refreshing the {Source} watchlist failed", source);
            return Failed(source, ex.Message);
        }
    }

    private WatchlistRefreshResult Failed(WatchlistSource source, string error)
    {
        _logger.LogWarning("Watchlist {Source} not loaded: {Error}", source, error);
        return new WatchlistRefreshResult(source, WatchlistRefreshStatus.FAILED, 0, error);
    }
}
```

Run Step 2's command. Expected: PASS.

- [ ] **Step 4: Downloader and cache**

Create `src/Aegis.Infrastructure/Screening/WatchlistDownloader.cs`:

```csharp
namespace Aegis.Infrastructure.Screening;

using Aegis.Application.Screening;
using Aegis.Modules.Screening.Domain;
using Microsoft.Extensions.Configuration;

/// <summary>Reads Screening:Sources:{UN|OFAC}:Path when set (offline installs, tests), else downloads :Url.</summary>
public sealed class WatchlistDownloader : IWatchlistDownloader
{
    public const string HttpClientName = "watchlists";
    private const long MaxBytes = 100 * 1024 * 1024;

    private static readonly Dictionary<WatchlistSource, string> DefaultUrls = new()
    {
        [WatchlistSource.UN] = "https://scsanctions.un.org/resources/xml/en/consolidated.xml",
        [WatchlistSource.OFAC] = "https://sanctionslistservice.ofac.treas.gov/api/PublicationPreview/exports/SDN.XML"
    };

    private readonly IHttpClientFactory _http;
    private readonly IConfiguration _config;

    public WatchlistDownloader(IHttpClientFactory http, IConfiguration config)
    {
        _http = http;
        _config = config;
    }

    public async Task<byte[]> DownloadAsync(WatchlistSource source, CancellationToken cancellationToken)
    {
        var path = _config[$"Screening:Sources:{source}:Path"];
        if (!string.IsNullOrWhiteSpace(path)) return await File.ReadAllBytesAsync(path, cancellationToken);

        var url = _config[$"Screening:Sources:{source}:Url"] ?? DefaultUrls[source];
        using var response = await _http.CreateClient(HttpClientName)
            .GetAsync(url, HttpCompletionOption.ResponseHeadersRead, cancellationToken);
        response.EnsureSuccessStatusCode();
        if (response.Content.Headers.ContentLength > MaxBytes)
            throw new InvalidOperationException($"{source} list is larger than {MaxBytes / 1024 / 1024} MB.");
        return await response.Content.ReadAsByteArrayAsync(cancellationToken);
    }
}
```

Create `src/Aegis.Infrastructure/Screening/GlobalWatchlistCache.cs`:

```csharp
namespace Aegis.Infrastructure.Screening;

using Aegis.Application.Screening;
using Aegis.Modules.Screening.Application;
using Aegis.Modules.Screening.Domain;
using Microsoft.Extensions.DependencyInjection;

/// <summary>
/// Singleton. Each call checks the newest load time (one cheap query) and rebuilds the index only when
/// a list changed, so every API instance picks up a refresh done by another.
/// </summary>
public sealed class GlobalWatchlistCache : IGlobalWatchlist
{
    private readonly IServiceScopeFactory _scopes;
    private readonly SemaphoreSlim _lock = new(1, 1);
    private (DateTimeOffset? LoadedAt, WatchlistIndex Index) _current = (null, new WatchlistIndex(Array.Empty<WatchlistSubject>()));

    public GlobalWatchlistCache(IServiceScopeFactory scopes) => _scopes = scopes;

    public async Task<WatchlistIndex> GetAsync(CancellationToken cancellationToken = default)
    {
        await using var scope = _scopes.CreateAsyncScope();
        var lists = scope.ServiceProvider.GetRequiredService<IWatchlistRepository>();
        var loadedAt = await lists.GetLastLoadedAtAsync(cancellationToken);
        if (loadedAt == _current.LoadedAt) return _current.Index;

        await _lock.WaitAsync(cancellationToken);
        try
        {
            if (loadedAt != _current.LoadedAt)
                _current = (loadedAt, new WatchlistIndex(await lists.ListSubjectsAsync(cancellationToken)));
            return _current.Index;
        }
        finally
        {
            _lock.Release();
        }
    }
}
```

Register in `DependencyInjection.cs`:

```csharp
services.AddHttpClient(Screening.WatchlistDownloader.HttpClientName, c => c.Timeout = TimeSpan.FromMinutes(3));
services.AddScoped<IWatchlistDownloader, Screening.WatchlistDownloader>();
services.AddScoped<IWatchlistRefresher, WatchlistRefresher>();
services.AddSingleton<IGlobalWatchlist, Screening.GlobalWatchlistCache>();
```

with `using Aegis.Application.Screening;`.

- [ ] **Step 5: Point the test factory at the fixtures and write the integration test**

In `AegisApiFactory` (`tests/Integration/Identity/AuthAndAuditTests.cs`), add to the settings dictionary:

```csharp
["Screening:Refresh:Enabled"] = "false",
["Screening:Sources:UN:Path"] = Path.Combine(AppContext.BaseDirectory, "Fixtures", "screening", "un-sample.xml"),
["Screening:Sources:OFAC:Path"] = Path.Combine(AppContext.BaseDirectory, "Fixtures", "screening", "ofac-sample.xml"),
["Risk:SyncScreeningFactorsOnStartup"] = "false",
```

(`Risk:SyncScreeningFactorsOnStartup` is read by Task 8; setting it now keeps the factory edit in one place.)

Create `tests/Integration/Screening/WatchlistRefreshTests.cs`:

```csharp
namespace Aegis.Tests.Integration.Screening;

using Aegis.Application.Screening;
using Aegis.Modules.Screening.Domain;
using Aegis.Tests.Integration.Alerts;
using Microsoft.Extensions.DependencyInjection;

public sealed class WatchlistRefreshTests : IAsyncLifetime
{
    private readonly AlertTestFixture _fx = new();

    public Task InitializeAsync() => _fx.InitializeAsync();
    public Task DisposeAsync() => _fx.DisposeAsync();

    [Fact]
    public async Task Fixture_lists_load_and_feed_the_global_index()
    {
        using var scope = _fx.Factory.Services.CreateScope();
        var refresher = scope.ServiceProvider.GetRequiredService<IWatchlistRefresher>();

        // The global tables are shared by every test class, so a previous run may already have loaded them.
        var un = await refresher.RefreshAsync(WatchlistSource.UN);
        var ofac = await refresher.RefreshAsync(WatchlistSource.OFAC);
        Assert.Contains(un.Status, new[] { WatchlistRefreshStatus.LOADED, WatchlistRefreshStatus.UNCHANGED });
        Assert.Equal(3, un.EntryCount);
        Assert.Equal(2, ofac.EntryCount);
        Assert.Equal(WatchlistRefreshStatus.UNCHANGED, (await refresher.RefreshAsync(WatchlistSource.UN)).Status);

        var index = await scope.ServiceProvider.GetRequiredService<IGlobalWatchlist>().GetAsync();
        Assert.Equal(5, index.Count);
    }
}
```

- [ ] **Step 6: Run the tests**

Run: `dotnet test tests/Unit && dotnet test tests/Integration --filter "FullyQualifiedName~Screening"`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add src tests
git commit -m "feat(screening): download, verify and replace UN and OFAC lists"
```

---

### Task 6: Screening service, screening alerts, daily rescreen job

**Files:**
- Create: `src/Aegis.Modules.Alerts/Domain/ScreeningAlertSource.cs`
- Modify: `src/Aegis.Modules.Alerts/Domain/Alert.cs` (`CreateFromScreening`)
- Modify: `src/Aegis.Modules.Alerts/Aegis.Modules.Alerts.csproj` only if needed (no Screening reference: the alert takes plain values)
- Create: `src/Aegis.Application/Screening/ScreeningAlertPolicy.cs`, `CustomerScreeningService.cs`, `IScreeningCandidateReader.cs`
- Create: `src/Aegis.Infrastructure/Screening/ScreeningCandidateReader.cs`, `WatchlistRefreshService.cs`
- Modify: `src/Aegis.Modules.Risk/Domain/CustomerRiskScore.cs` (`RiskTriggers.ScreeningConfirmed`)
- Modify: `src/Aegis.Modules.Audit/Domain/AuditEventTypes.cs` (`SCREENING_LIST_UPLOADED`)
- Modify: `src/Aegis.Infrastructure/Dashboard/DashboardQueries.cs` (name the screening "rule" in top rules)
- Modify: `src/Aegis.Infrastructure/DependencyInjection.cs`
- Test: `tests/Unit/Alerts/ScreeningAlertTests.cs`, `tests/Unit/Screening/ScreeningAlertPolicyTests.cs`

**Interfaces:**
- Consumes: Tasks 1–5; `IAlertRepository`, `IAuditWriter`, `IUnitOfWork`, `ICustomerRiskService`, `RiskActor`, `IRiskBatchSource`, `RiskBatchSchedule`.
- Produces:
  - `ScreeningAlertSource.RuleId` (`Guid` `5c3e0000-0000-4000-8000-000000000001`), `ScreeningAlertSource.RuleCode = "SCREENING_MATCH"` (namespace `Aegis.Modules.Alerts.Domain`)
  - `static Alert Alert.CreateFromScreening(TenantId tenantId, Guid customerId, Guid matchId, AlertSeverity severity, int riskScore, AlertEvidence evidence)`, deduplication key `"{tenant:D}:screening:{matchId:D}"`
  - `static (AlertSeverity Severity, int RiskScore, string Title) ScreeningAlertPolicy.For(WatchlistCategory category)`; `static AlertEvidence ScreeningAlertPolicy.Evidence(ScreeningMatch match, string title)`
  - `record ScreeningOptions(double MatchThreshold = WatchlistIndex.DefaultThreshold)`
  - `interface IScreeningCandidateReader { Task<ScreeningCandidate?> GetAsync(TenantId, Guid customerId, CancellationToken); Task<IReadOnlyList<ScreeningCandidate>> ListAsync(TenantId, Guid? afterId, int take, CancellationToken); }`
  - `enum ScreeningDecision { CONFIRM, FALSE_POSITIVE }`, `record ScreeningOutcome(int Screened, IReadOnlyList<Guid> NewMatchIds)`
  - `interface ICustomerScreeningService` with `ScreenCustomerAsync`, `TryScreenCustomerAsync`, `ScreenTenantAsync`, `ReviewAsync` (signatures in Step 3)
  - `RiskTriggers.ScreeningConfirmed = "SCREENING_CONFIRMED"`; `AuditEventTypes.SCREENING_LIST_UPLOADED`

- [ ] **Step 1: Write the failing unit tests**

Create `tests/Unit/Alerts/ScreeningAlertTests.cs`:

```csharp
namespace Aegis.Tests.Unit.Alerts;

using Aegis.Modules.Alerts.Domain;
using Aegis.Shared.Domain;

public sealed class ScreeningAlertTests
{
    [Fact]
    public void Screening_alert_uses_the_fixed_rule_and_a_per_match_dedupe_key()
    {
        var tenant = new TenantId(Guid.NewGuid());
        var customerId = Guid.NewGuid();
        var matchId = Guid.NewGuid();

        var alert = Alert.CreateFromScreening(tenant, customerId, matchId, AlertSeverity.CRITICAL, 95, new AlertEvidence { RuleName = "Confirmed sanctions match" });

        Assert.Equal(ScreeningAlertSource.RuleId, alert.RuleId);
        Assert.Equal(ScreeningAlertSource.RuleId, alert.RuleVersionId);
        Assert.Equal(FocusType.CUSTOMER, alert.FocusType);
        Assert.Equal(customerId.ToString(), alert.FocusEntityId);
        Assert.Equal(AlertSeverity.CRITICAL, alert.Severity);
        Assert.Equal($"{tenant.Value:D}:screening:{matchId:D}", alert.DeduplicationKey);
    }
}
```

Create `tests/Unit/Screening/ScreeningAlertPolicyTests.cs`:

```csharp
namespace Aegis.Tests.Unit.Screening;

using Aegis.Application.Screening;
using Aegis.Modules.Alerts.Domain;
using Aegis.Modules.Screening.Domain;
using Aegis.Shared.Domain;

public sealed class ScreeningAlertPolicyTests
{
    [Theory]
    [InlineData(WatchlistCategory.SANCTIONS, AlertSeverity.CRITICAL, 95, "Confirmed sanctions match")]
    [InlineData(WatchlistCategory.WATCHLIST, AlertSeverity.CRITICAL, 90, "Confirmed watchlist match")]
    [InlineData(WatchlistCategory.PEP, AlertSeverity.HIGH, 70, "Confirmed PEP match")]
    public void Severity_follows_the_category(WatchlistCategory category, AlertSeverity severity, int risk, string title)
        => Assert.Equal((severity, risk, title), ScreeningAlertPolicy.For(category));

    [Fact]
    public void Evidence_carries_what_the_console_needs()
    {
        var subject = new WatchlistSubject(WatchlistSource.UN, "QDi.901", WatchlistCategory.SANCTIONS, SubjectKind.INDIVIDUAL,
            new[] { "ZEPHANIAH OKOTH MBWANA" }, Array.Empty<BirthYearRange>(), Array.Empty<string>(), null);
        var match = ScreeningMatch.Create(new TenantId(Guid.NewGuid()),
            new ScreeningCandidate(Guid.NewGuid(), "Zephaniah Mbwana", SubjectKind.INDIVIDUAL, null),
            new PotentialMatch(subject, "ZEPHANIAH OKOTH MBWANA", 0.9333, true), ScreeningTriggers.Manual);
        match.Confirm("user-1", "Same ID number", Guid.NewGuid());

        var evidence = ScreeningAlertPolicy.Evidence(match, "Confirmed sanctions match");

        Assert.Equal("Confirmed sanctions match", evidence.RuleName);
        Assert.Equal("UN", evidence.EvaluatedValues["list"]);
        Assert.Equal("SANCTIONS", evidence.EvaluatedValues["category"]);
        Assert.Equal("ZEPHANIAH OKOTH MBWANA", evidence.EvaluatedValues["matchedName"]);
        Assert.Equal(0.9333, evidence.EvaluatedValues["matchScore"]);
        Assert.Equal(true, evidence.EvaluatedValues["birthYearMatched"]);
        Assert.Equal(ScreeningAlertSource.RuleCode, evidence.AdditionalContext["ruleCode"]);
        Assert.Equal(match.Id.ToString("D"), evidence.AdditionalContext["screeningMatchId"]);
        Assert.Equal("Same ID number", evidence.AdditionalContext["reviewReason"]);
        Assert.Empty(evidence.TransactionIds);
    }
}
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `dotnet test tests/Unit --filter "FullyQualifiedName~ScreeningAlert"`
Expected: build fails.

- [ ] **Step 3: Implement the alert source, policy and service**

Create `src/Aegis.Modules.Alerts/Domain/ScreeningAlertSource.cs`:

```csharp
namespace Aegis.Modules.Alerts.Domain;

/// <summary>Alerts raised by confirmed screening matches have no AML rule; they all carry this fixed rule id.</summary>
public static class ScreeningAlertSource
{
    public static readonly Guid RuleId = new("5c3e0000-0000-4000-8000-000000000001");
    public const string RuleCode = "SCREENING_MATCH";
}
```

Add to `Alert.cs` (the optional id lets the service record the alert id on the match before the alert is built from the match's evidence):

```csharp
public static Alert CreateFromScreening(
    TenantId tenantId, Guid customerId, Guid matchId, AlertSeverity severity, int riskScore, AlertEvidence evidence,
    Guid? alertId = null)
{
    var alert = Create(
        tenantId, ScreeningAlertSource.RuleId, ScreeningAlertSource.RuleId, FocusType.CUSTOMER, customerId.ToString(),
        severity, riskScore, evidence, $"{tenantId.Value:D}:screening:{matchId:D}");
    if (alertId is { } id) alert.Id = id;
    return alert;
}
```

Add `public const string ScreeningConfirmed = "SCREENING_CONFIRMED";` to `RiskTriggers` and `public const string SCREENING_LIST_UPLOADED = nameof(SCREENING_LIST_UPLOADED);` after `SCREENING_MATCH_RESOLVED` in `AuditEventTypes`.

Create `src/Aegis.Application/Screening/ScreeningAlertPolicy.cs`:

```csharp
namespace Aegis.Application.Screening;

using Aegis.Modules.Alerts.Domain;
using Aegis.Modules.Screening.Domain;
using Aegis.Shared.Domain;

public sealed record ScreeningOptions(double MatchThreshold = WatchlistIndex.DefaultThreshold);

public static class ScreeningAlertPolicy
{
    /// <summary>PEP status calls for enhanced due diligence, not a freeze, so it is HIGH rather than CRITICAL.</summary>
    public static (AlertSeverity Severity, int RiskScore, string Title) For(WatchlistCategory category) => category switch
    {
        WatchlistCategory.SANCTIONS => (AlertSeverity.CRITICAL, 95, "Confirmed sanctions match"),
        WatchlistCategory.WATCHLIST => (AlertSeverity.CRITICAL, 90, "Confirmed watchlist match"),
        WatchlistCategory.PEP => (AlertSeverity.HIGH, 70, "Confirmed PEP match"),
        _ => throw new ArgumentOutOfRangeException(nameof(category), category, null)
    };

    public static AlertEvidence Evidence(ScreeningMatch match, string title) => new()
    {
        RuleName = title,
        RuleVersionNumber = 1,
        EvaluatedValues = new Dictionary<string, object>
        {
            ["list"] = match.Source.ToString(),
            ["category"] = match.Category.ToString(),
            ["matchedName"] = match.MatchedName,
            ["customerName"] = match.CustomerName,
            ["matchScore"] = match.Score,
            ["externalId"] = match.ExternalId,
            ["birthYearMatched"] = match.BirthYearMatched
        },
        AdditionalContext = new Dictionary<string, object>
        {
            ["ruleCode"] = ScreeningAlertSource.RuleCode,
            ["screeningMatchId"] = match.Id.ToString("D"),
            ["reviewReason"] = match.ReviewReason ?? ""
        }
    };
}
```

Create `src/Aegis.Application/Screening/IScreeningCandidateReader.cs`:

```csharp
namespace Aegis.Application.Screening;

using Aegis.Modules.Screening.Domain;
using Aegis.Shared.Domain;

public interface IScreeningCandidateReader
{
    Task<ScreeningCandidate?> GetAsync(TenantId tenantId, Guid customerId, CancellationToken cancellationToken = default);

    /// <summary>Customers ordered by id, after <paramref name="afterId"/>.</summary>
    Task<IReadOnlyList<ScreeningCandidate>> ListAsync(TenantId tenantId, Guid? afterId, int take, CancellationToken cancellationToken = default);
}
```

Create `src/Aegis.Application/Screening/CustomerScreeningService.cs`:

```csharp
namespace Aegis.Application.Screening;

using Aegis.Application.Risk;
using Aegis.Modules.Alerts.Application;
using Aegis.Modules.Alerts.Domain;
using Aegis.Modules.Audit.Application;
using Aegis.Modules.Audit.Domain;
using Aegis.Modules.Risk.Domain;
using Aegis.Modules.Screening.Application;
using Aegis.Modules.Screening.Domain;
using Aegis.Shared.Domain;
using Aegis.Shared.Persistence;
using Microsoft.Extensions.Logging;

public enum ScreeningDecision { CONFIRM, FALSE_POSITIVE }

public sealed record ScreeningOutcome(int Screened, IReadOnlyList<Guid> NewMatchIds);

public interface ICustomerScreeningService
{
    /// <returns>Null when the customer does not exist in the tenant.</returns>
    Task<ScreeningOutcome?> ScreenCustomerAsync(
        TenantId tenantId, Guid customerId, string trigger, RiskActor actor, CancellationToken cancellationToken = default);

    /// <summary><see cref="ScreenCustomerAsync"/> for side-effect triggers: a failure is logged and never fails the caller.</summary>
    Task TryScreenCustomerAsync(
        TenantId tenantId, Guid customerId, string trigger, RiskActor actor, CancellationToken cancellationToken = default);

    Task<ScreeningOutcome> ScreenTenantAsync(TenantId tenantId, string trigger, RiskActor actor, CancellationToken cancellationToken = default);

    /// <returns>Null when the match does not exist in the tenant.</returns>
    /// <exception cref="ArgumentException">The reason is blank.</exception>
    /// <exception cref="InvalidOperationException">The match was already reviewed.</exception>
    Task<ScreeningMatch?> ReviewAsync(
        TenantId tenantId, Guid matchId, ScreeningDecision decision, string reason, RiskActor actor,
        CancellationToken cancellationToken = default);
}

public sealed class CustomerScreeningService : ICustomerScreeningService
{
    private const int BatchSize = 500;

    private readonly IScreeningCandidateReader _candidates;
    private readonly IGlobalWatchlist _global;
    private readonly ITenantWatchlistRepository _tenantLists;
    private readonly IScreeningMatchRepository _matches;
    private readonly IAlertRepository _alerts;
    private readonly IAuditWriter _audit;
    private readonly IUnitOfWork _uow;
    private readonly ICustomerRiskService _risk;
    private readonly ScreeningOptions _options;
    private readonly ILogger<CustomerScreeningService> _logger;

    public CustomerScreeningService(
        IScreeningCandidateReader candidates,
        IGlobalWatchlist global,
        ITenantWatchlistRepository tenantLists,
        IScreeningMatchRepository matches,
        IAlertRepository alerts,
        IAuditWriter audit,
        IUnitOfWork uow,
        ICustomerRiskService risk,
        ScreeningOptions options,
        ILogger<CustomerScreeningService> logger)
    {
        _candidates = candidates;
        _global = global;
        _tenantLists = tenantLists;
        _matches = matches;
        _alerts = alerts;
        _audit = audit;
        _uow = uow;
        _risk = risk;
        _options = options;
        _logger = logger;
    }

    public async Task<ScreeningOutcome?> ScreenCustomerAsync(
        TenantId tenantId, Guid customerId, string trigger, RiskActor actor, CancellationToken cancellationToken = default)
    {
        var candidate = await _candidates.GetAsync(tenantId, customerId, cancellationToken);
        if (candidate is null) return null;

        var created = await ScreenAsync(tenantId, candidate, await ListsAsync(tenantId, cancellationToken), trigger, actor, cancellationToken);
        await _uow.SaveChangesAsync(cancellationToken);
        return new ScreeningOutcome(1, created);
    }

    public async Task TryScreenCustomerAsync(
        TenantId tenantId, Guid customerId, string trigger, RiskActor actor, CancellationToken cancellationToken = default)
    {
        try
        {
            await ScreenCustomerAsync(tenantId, customerId, trigger, actor, cancellationToken);
        }
        catch (Exception ex) when (ex is not OperationCanceledException)
        {
            _logger.LogError(ex, "Screening ({Trigger}) failed for customer {CustomerId}", trigger, customerId);
        }
    }

    public async Task<ScreeningOutcome> ScreenTenantAsync(
        TenantId tenantId, string trigger, RiskActor actor, CancellationToken cancellationToken = default)
    {
        var lists = await ListsAsync(tenantId, cancellationToken);
        var created = new List<Guid>();
        var screened = 0;
        Guid? after = null;
        while (true)
        {
            var batch = await _candidates.ListAsync(tenantId, after, BatchSize, cancellationToken);
            if (batch.Count == 0) break;
            foreach (var candidate in batch)
                created.AddRange(await ScreenAsync(tenantId, candidate, lists, trigger, actor, cancellationToken));
            await _uow.SaveChangesAsync(cancellationToken);
            screened += batch.Count;
            after = batch[^1].CustomerId;
        }
        return new ScreeningOutcome(screened, created);
    }

    public async Task<ScreeningMatch?> ReviewAsync(
        TenantId tenantId, Guid matchId, ScreeningDecision decision, string reason, RiskActor actor,
        CancellationToken cancellationToken = default)
    {
        var match = await _matches.GetAsync(tenantId, matchId, cancellationToken);
        if (match is null) return null;
        var before = match.Status;

        if (decision == ScreeningDecision.CONFIRM)
        {
            var (severity, riskScore, title) = ScreeningAlertPolicy.For(match.Category);
            var alertId = Guid.NewGuid();
            match.Confirm(actor.Id, reason, alertId);
            var alert = Alert.CreateFromScreening(
                tenantId, match.CustomerId, match.Id, severity, riskScore, ScreeningAlertPolicy.Evidence(match, title), alertId);
            await _alerts.AddAsync(alert, cancellationToken);
            await _audit.AppendAsync(AuditEvent.Create(
                tenantId.Value, AuditEventTypes.ALERT_CREATED, nameof(Alert), alert.Id.ToString(), actor.Id, actor.Role, null,
                AuditPayload.Json(new { ruleCode = ScreeningAlertSource.RuleCode, severity = severity.ToString(), screeningMatchId = match.Id }),
                $"{title}: {match.MatchedName}", actor.CorrelationId), cancellationToken);
        }
        else
        {
            match.MarkFalsePositive(actor.Id, reason);
        }

        await _audit.AppendAsync(AuditEvent.Create(
            tenantId.Value, AuditEventTypes.SCREENING_MATCH_RESOLVED, nameof(ScreeningMatch), match.Id.ToString(), actor.Id, actor.Role,
            AuditPayload.Json(new { status = before.ToString() }),
            AuditPayload.Json(new { status = match.Status.ToString(), reason = match.ReviewReason, alertId = match.AlertId }),
            decision == ScreeningDecision.CONFIRM ? "Screening match confirmed" : "Screening match marked false positive",
            actor.CorrelationId), cancellationToken);
        await _uow.SaveChangesAsync(cancellationToken);

        if (decision == ScreeningDecision.CONFIRM)
            await _risk.TryRecalculateAsync(tenantId, match.CustomerId, RiskTriggers.ScreeningConfirmed, actor, cancellationToken);
        return match;
    }

    private async Task<WatchlistIndex[]> ListsAsync(TenantId tenantId, CancellationToken cancellationToken)
        => new[]
        {
            await _global.GetAsync(cancellationToken),
            new WatchlistIndex(await _tenantLists.ListSubjectsAsync(tenantId, cancellationToken))
        };

    private async Task<IReadOnlyList<Guid>> ScreenAsync(
        TenantId tenantId, ScreeningCandidate candidate, WatchlistIndex[] lists, string trigger, RiskActor actor,
        CancellationToken cancellationToken)
    {
        var found = lists.SelectMany(l => l.Screen(candidate, _options.MatchThreshold)).ToList();
        if (found.Count == 0) return Array.Empty<Guid>();

        var known = await _matches.ListKeysAsync(tenantId, candidate.CustomerId, cancellationToken);
        var created = new List<Guid>();
        foreach (var potential in found)
        {
            if (known.Contains((potential.Subject.Source, potential.Subject.ExternalId))) continue;
            var match = ScreeningMatch.Create(tenantId, candidate, potential, trigger);
            await _matches.AddAsync(match, cancellationToken);
            await _audit.AppendAsync(AuditEvent.Create(
                tenantId.Value, AuditEventTypes.SCREENING_MATCH_CREATED, nameof(ScreeningMatch), match.Id.ToString(), actor.Id, actor.Role, null,
                AuditPayload.Json(new
                {
                    customerId = candidate.CustomerId,
                    source = potential.Subject.Source.ToString(),
                    externalId = potential.Subject.ExternalId,
                    matchedName = potential.MatchedName,
                    score = potential.Score
                }),
                $"Possible {potential.Subject.Category} match: {potential.MatchedName}", actor.CorrelationId), cancellationToken);
            created.Add(match.Id);
        }
        return created;
    }
}
```

`Confirm` throws before anything is added when the reason is blank or the match is already reviewed. Two reviewers confirming at once both pass `Confirm` in memory, but the alert's unique `(tenant_id, DeduplicationKey)` index makes the second `SaveChangesAsync` throw `UniqueConstraintViolationException`, which the controller turns into 409.

- [ ] **Step 4: Candidate reader, dashboard naming, registrations**

Create `src/Aegis.Infrastructure/Screening/ScreeningCandidateReader.cs`:

```csharp
namespace Aegis.Infrastructure.Screening;

using Aegis.Application.Screening;
using Aegis.Infrastructure.Persistence;
using Aegis.Modules.Customers.Domain;
using Aegis.Modules.Screening.Domain;
using Aegis.Shared.Domain;
using Microsoft.EntityFrameworkCore;

public sealed class ScreeningCandidateReader : IScreeningCandidateReader
{
    private readonly AegisDbContext _db;

    public ScreeningCandidateReader(AegisDbContext db) => _db = db;

    public async Task<ScreeningCandidate?> GetAsync(TenantId tenantId, Guid customerId, CancellationToken cancellationToken = default)
        => (await Query(tenantId).Where(c => c.Id == customerId).Take(1).ToListAsync(cancellationToken))
            .Select(ToCandidate).FirstOrDefault();

    public async Task<IReadOnlyList<ScreeningCandidate>> ListAsync(TenantId tenantId, Guid? afterId, int take, CancellationToken cancellationToken = default)
    {
        var query = Query(tenantId);
        if (afterId is Guid after) query = query.Where(c => c.Id.CompareTo(after) > 0);
        var rows = await query.OrderBy(c => c.Id).Take(take).ToListAsync(cancellationToken);
        return rows.Select(ToCandidate).ToList();
    }

    private IQueryable<Customer> Query(TenantId tenantId) => _db.Customers.AsNoTracking().Where(c => c.TenantId == tenantId);

    private static ScreeningCandidate ToCandidate(Customer c) => c.Type == CustomerType.INDIVIDUAL
        ? new ScreeningCandidate(c.Id, $"{c.FirstName} {c.LastName}".Trim(), SubjectKind.INDIVIDUAL, c.DateOfBirth)
        : new ScreeningCandidate(c.Id, c.LegalName ?? "", SubjectKind.ENTITY, null);
}
```

In `DashboardQueries.cs`, where the top-rules fallback builds `"Unknown rule"`, name the screening rule instead:

```csharp
: r.RuleId == ScreeningAlertSource.RuleId
    ? new RuleAlertStats(r.RuleId, "", "Confirmed screening matches", r.Total, r.Dismissed, DashboardMath.Rate(r.Dismissed, r.Total))
    : new RuleAlertStats(r.RuleId, "", "Unknown rule", r.Total, r.Dismissed, DashboardMath.Rate(r.Dismissed, r.Total)))
```

An empty `RuleCode` already stops the console from linking the row to `/rules/{id}`.

Register in `DependencyInjection.cs`:

```csharp
services.AddSingleton(sp => new ScreeningOptions(
    sp.GetRequiredService<IConfiguration>().GetValue("Screening:MatchThreshold", WatchlistIndex.DefaultThreshold)));
services.AddScoped<IScreeningCandidateReader, Screening.ScreeningCandidateReader>();
services.AddScoped<ICustomerScreeningService, CustomerScreeningService>();
services.AddHostedService<Screening.WatchlistRefreshService>();
```

- [ ] **Step 5: Daily refresh and rescreen job**

Create `src/Aegis.Infrastructure/Screening/WatchlistRefreshService.cs`:

```csharp
namespace Aegis.Infrastructure.Screening;

using Aegis.Application.Risk;
using Aegis.Application.Screening;
using Aegis.Modules.Screening.Application;
using Aegis.Modules.Screening.Domain;
using Aegis.Shared.Domain;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;

/// <summary>
/// Loads any list that has never been loaded at startup, then refreshes UN and OFAC daily and rescreens every
/// customer of every non-suspended tenant when a list changed.
/// Settings: Screening:Refresh:Enabled (default true) and Screening:Refresh:HourUtc (default 1).
/// </summary>
public sealed class WatchlistRefreshService : BackgroundService
{
    private readonly IServiceScopeFactory _scopes;
    private readonly IConfiguration _config;
    private readonly ILogger<WatchlistRefreshService> _logger;

    public WatchlistRefreshService(IServiceScopeFactory scopes, IConfiguration config, ILogger<WatchlistRefreshService> logger)
    {
        _scopes = scopes;
        _config = config;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        if (!_config.GetValue("Screening:Refresh:Enabled", true))
        {
            _logger.LogInformation("Watchlist refresh disabled");
            return;
        }

        await RunOnceAsync(onlyMissing: true, stoppingToken);
        var hourUtc = _config.GetValue("Screening:Refresh:HourUtc", 1);
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await Task.Delay(RiskBatchSchedule.NextRun(DateTimeOffset.UtcNow, hourUtc) - DateTimeOffset.UtcNow, stoppingToken);
                await RunOnceAsync(onlyMissing: false, stoppingToken);
            }
            catch (OperationCanceledException) when (stoppingToken.IsCancellationRequested)
            {
                return;
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Watchlist refresh failed");
            }
        }
    }

    private async Task RunOnceAsync(bool onlyMissing, CancellationToken ct)
    {
        var changed = false;
        await using (var scope = _scopes.CreateAsyncScope())
        {
            var refresher = scope.ServiceProvider.GetRequiredService<IWatchlistRefresher>();
            var lists = scope.ServiceProvider.GetRequiredService<IWatchlistRepository>();
            foreach (var source in refresher.Sources)
            {
                if (onlyMissing && await lists.GetLatestLoadAsync(source, ct) is not null) continue;
                changed |= (await refresher.RefreshAsync(source, ct)).Status == WatchlistRefreshStatus.LOADED;
            }
        }
        if (!changed) return;

        IReadOnlyList<Guid> tenantIds;
        await using (var scope = _scopes.CreateAsyncScope())
        {
            tenantIds = await scope.ServiceProvider.GetRequiredService<IRiskBatchSource>().ListScorableTenantIdsAsync(ct);
        }

        foreach (var tenantId in tenantIds)
        {
            await using var scope = _scopes.CreateAsyncScope();
            try
            {
                var outcome = await scope.ServiceProvider.GetRequiredService<ICustomerScreeningService>()
                    .ScreenTenantAsync(new TenantId(tenantId), ScreeningTriggers.ListUpdated, RiskActor.System, ct);
                _logger.LogInformation("Rescreened {Count} customers for tenant {TenantId}: {Matches} new matches",
                    outcome.Screened, tenantId, outcome.NewMatchIds.Count);
            }
            catch (Exception ex) when (ex is not OperationCanceledException)
            {
                _logger.LogError(ex, "Rescreen failed for tenant {TenantId}", tenantId);
            }
        }
    }
}
```

- [ ] **Step 6: Run the tests**

Run: `dotnet build && dotnet test tests/Unit && dotnet test tests/Integration`
Expected: PASS (the service is exercised end to end through the API in Task 7).

- [ ] **Step 7: Commit**

```bash
git add src tests/Unit
git commit -m "feat(screening): screen customers, review matches, raise screening alerts, daily rescreen"
```

---

### Task 7: Screening API and permissions

**Files:**
- Modify: `src/Aegis.Shared/Security/Permissions.cs`
- Create: `src/Aegis.Api/Controllers/ScreeningController.cs`
- Modify: `src/Aegis.Api/Controllers/CustomersController.cs` (date of birth, screen on create)
- Modify: `src/Aegis.Application/Dashboard/DashboardMetrics.cs`, `src/Aegis.Infrastructure/Dashboard/DashboardQueries.cs`, `src/Aegis.Api/Controllers/DashboardController.cs` (pending review count)
- Test: `tests/Unit/Security/PermissionsTests.cs` (extend if it lists grants), `tests/Integration/Screening/ScreeningApiTests.cs`

**Interfaces:**
- Consumes: Task 6 service; Task 4 repositories; Task 5 refresher.
- Produces:
  - `Permissions.ScreeningRead = "screening.read"` (Viewer and up), `ScreeningReview = "screening.review"` (Reviewer, Admin), `ScreeningManage = "screening.manage"` (Admin)
  - Endpoints (all under `/api/v1`):
    - `GET screening/lists` (read): `{ sources: [{ source, loadedAt, publishedAt, entryCount }], tenantList: { entryCount, uploadedAt, uploadedBy } }`
    - `POST screening/lists/refresh` (manage): `{ results: [{ source, status, entryCount, error }], newMatches }`
    - `PUT screening/tenant-list` (manage, multipart `file`, at most 5 MB): `{ entryCount, newMatches }`, or 400 `{ errors: [...] }`
    - `GET screening/matches?status=&customerId=&page=&pageSize=` (read): `{ items, page, pageSize, totalCount }`
    - `GET screening/matches/{id}` (read): `{ match, subject, reviewedBy, reviewReason, alertId }`
    - `POST screening/matches/{id}/confirm` and `/dismiss` (review), body `{ reason }`: the summary item; 400 blank reason; 409 already reviewed
    - `POST customers/{id}/screen` (`customer.write`): `{ newMatches, matchIds }`
  - `POST customers` accepts optional `dateOfBirth` (`yyyy-MM-dd`, not in the future); customer responses include `dateOfBirth`.
  - Dashboard response gains `screening: { pendingReview }` (null without `screening.read`).

- [ ] **Step 1: Write the failing integration tests**

Create `tests/Integration/Screening/ScreeningApiTests.cs`:

```csharp
namespace Aegis.Tests.Integration.Screening;

using System.Net;
using System.Net.Http.Json;
using System.Text;
using System.Text.Json;
using Aegis.Shared.Security;
using Aegis.Tests.Integration.Alerts;

public sealed class ScreeningApiTests : IAsyncLifetime
{
    private readonly AlertTestFixture _fx = new();

    public async Task InitializeAsync()
    {
        await _fx.InitializeAsync();
        (await _fx.AdminClient.PostAsync("/api/v1/screening/lists/refresh", null)).EnsureSuccessStatusCode();
    }

    public Task DisposeAsync() => _fx.DisposeAsync();

    [Fact]
    public async Task Lists_report_both_global_sources()
    {
        var body = await _fx.AdminClient.GetFromJsonAsync<JsonElement>("/api/v1/screening/lists");
        var sources = body.GetProperty("sources").EnumerateArray().ToDictionary(
            s => s.GetProperty("source").GetString()!, s => s.GetProperty("entryCount").GetInt32());
        Assert.Equal(3, sources["UN"]);
        Assert.Equal(2, sources["OFAC"]);
        Assert.Equal(0, body.GetProperty("tenantList").GetProperty("entryCount").GetInt32());
    }

    [Fact]
    public async Task Creating_a_listed_customer_records_a_pending_match()
    {
        var customerId = await CreateIndividualAsync("Zephaniah", "Mbwana", "1978-04-28");

        var match = await SingleMatchAsync(customerId);

        Assert.Equal("PENDING_REVIEW", match.GetProperty("status").GetString());
        Assert.Equal("UN", match.GetProperty("source").GetString());
        Assert.Equal("SANCTIONS", match.GetProperty("category").GetString());
        Assert.True(match.GetProperty("birthYearMatched").GetBoolean());
        Assert.True(match.GetProperty("score").GetDouble() >= 0.88);
        Assert.Equal("CUSTOMER_CREATED", match.GetProperty("trigger").GetString());
    }

    [Fact]
    public async Task Birth_year_outside_the_listed_range_is_not_a_match()
    {
        var customerId = await CreateIndividualAsync("Zephaniah", "Mbwana", "1990-01-01");
        Assert.Equal(0, (await MatchesAsync(customerId)).GetProperty("totalCount").GetInt32());
    }

    [Fact]
    public async Task Future_date_of_birth_is_rejected()
    {
        var response = await _fx.AdminClient.PostAsJsonAsync("/api/v1/customers", new
        {
            type = "INDIVIDUAL", country = "KE", firstName = "Future", lastName = "Person",
            dateOfBirth = DateTime.UtcNow.AddDays(2).ToString("yyyy-MM-dd")
        });
        Assert.Equal(HttpStatusCode.BadRequest, response.StatusCode);
    }

    [Fact]
    public async Task Confirming_raises_a_critical_alert_once()
    {
        var customerId = await CreateIndividualAsync("Zephaniah", "Mbwana", null);
        var matchId = (await SingleMatchAsync(customerId)).GetProperty("id").GetGuid();

        var confirm = await _fx.AdminClient.PostAsJsonAsync($"/api/v1/screening/matches/{matchId}/confirm", new { reason = "Same national ID" });
        Assert.Equal(HttpStatusCode.OK, confirm.StatusCode);

        var detail = await _fx.AdminClient.GetFromJsonAsync<JsonElement>($"/api/v1/screening/matches/{matchId}");
        Assert.Equal("CONFIRMED", detail.GetProperty("match").GetProperty("status").GetString());
        Assert.Equal("Same national ID", detail.GetProperty("reviewReason").GetString());
        var alertId = detail.GetProperty("alertId").GetGuid();

        var alert = await _fx.AdminClient.GetFromJsonAsync<JsonElement>($"/api/v1/alerts/{alertId}");
        Assert.Equal("CRITICAL", alert.GetProperty("severity").GetString());
        Assert.Equal("Confirmed sanctions match", alert.GetProperty("ruleName").GetString());

        var again = await _fx.AdminClient.PostAsJsonAsync($"/api/v1/screening/matches/{matchId}/confirm", new { reason = "Twice" });
        Assert.Equal(HttpStatusCode.Conflict, again.StatusCode);
    }

    [Fact]
    public async Task Dismissing_needs_a_reason_and_a_rescreen_does_not_recreate_the_match()
    {
        var customerId = await CreateIndividualAsync("Zephaniah", "Mbwana", null);
        var matchId = (await SingleMatchAsync(customerId)).GetProperty("id").GetGuid();

        var blank = await _fx.AdminClient.PostAsJsonAsync($"/api/v1/screening/matches/{matchId}/dismiss", new { reason = " " });
        Assert.Equal(HttpStatusCode.BadRequest, blank.StatusCode);

        var dismiss = await _fx.AdminClient.PostAsJsonAsync($"/api/v1/screening/matches/{matchId}/dismiss", new { reason = "Different person: born 1990" });
        Assert.Equal(HttpStatusCode.OK, dismiss.StatusCode);

        var rescreen = await _fx.AdminClient.PostAsync($"/api/v1/customers/{customerId}/screen", null);
        Assert.Equal(0, (await rescreen.Content.ReadFromJsonAsync<JsonElement>()).GetProperty("newMatches").GetInt32());
        Assert.Equal("FALSE_POSITIVE", (await SingleMatchAsync(customerId)).GetProperty("status").GetString());
    }

    [Fact]
    public async Task Viewers_read_reviewers_decide_admins_manage_lists()
    {
        var customerId = await CreateIndividualAsync("Zephaniah", "Mbwana", null);
        var matchId = (await SingleMatchAsync(customerId)).GetProperty("id").GetGuid();
        var viewer = await _fx.CreateUserClientAsync($"viewer@{_fx.Slug}.test", new[] { RoleNames.Viewer });
        var analyst = await _fx.CreateUserClientAsync($"analyst@{_fx.Slug}.test", new[] { RoleNames.Analyst });
        var reviewer = await _fx.CreateUserClientAsync($"reviewer@{_fx.Slug}.test", new[] { RoleNames.Reviewer });

        Assert.Equal(HttpStatusCode.OK, (await viewer.GetAsync("/api/v1/screening/matches")).StatusCode);
        Assert.Equal(HttpStatusCode.Forbidden, (await viewer.PostAsJsonAsync($"/api/v1/screening/matches/{matchId}/confirm", new { reason = "x" })).StatusCode);
        Assert.Equal(HttpStatusCode.Forbidden, (await analyst.PostAsJsonAsync($"/api/v1/screening/matches/{matchId}/confirm", new { reason = "x" })).StatusCode);
        Assert.Equal(HttpStatusCode.Forbidden, (await reviewer.PostAsync("/api/v1/screening/lists/refresh", null)).StatusCode);
        Assert.Equal(HttpStatusCode.OK, (await reviewer.PostAsJsonAsync($"/api/v1/screening/matches/{matchId}/dismiss", new { reason = "Different person" })).StatusCode);
    }

    [Fact]
    public async Task Uploaded_list_screens_existing_customers()
    {
        var customerId = await CreateIndividualAsync("Grace", "Akinyi Otieno", null);

        var upload = await UploadAsync(
            "reference,name,category,kind\nPEP-1,Grace Akinyi Otieno,PEP,INDIVIDUAL\n");

        Assert.Equal(HttpStatusCode.OK, upload.StatusCode);
        var body = await upload.Content.ReadFromJsonAsync<JsonElement>();
        Assert.Equal(1, body.GetProperty("entryCount").GetInt32());
        Assert.Equal(1, body.GetProperty("newMatches").GetInt32());
        var match = await SingleMatchAsync(customerId);
        Assert.Equal("TENANT", match.GetProperty("source").GetString());
        Assert.Equal("PEP", match.GetProperty("category").GetString());
    }

    [Fact]
    public async Task Invalid_list_is_rejected_with_line_errors()
    {
        var upload = await UploadAsync("reference,name,category,kind\nX-1,Someone,FRIEND,INDIVIDUAL\n");
        Assert.Equal(HttpStatusCode.BadRequest, upload.StatusCode);
        var errors = (await upload.Content.ReadFromJsonAsync<JsonElement>()).GetProperty("errors");
        Assert.StartsWith("Line 2:", errors[0].GetString());
    }

    [Fact]
    public async Task Another_tenant_cannot_see_or_review_the_match()
    {
        var customerId = await CreateIndividualAsync("Zephaniah", "Mbwana", null);
        var matchId = (await SingleMatchAsync(customerId)).GetProperty("id").GetGuid();
        var other = new AlertTestFixture();
        await other.InitializeAsync();
        try
        {
            Assert.Equal(HttpStatusCode.NotFound, (await other.AdminClient.GetAsync($"/api/v1/screening/matches/{matchId}")).StatusCode);
            Assert.Equal(HttpStatusCode.NotFound,
                (await other.AdminClient.PostAsJsonAsync($"/api/v1/screening/matches/{matchId}/dismiss", new { reason = "x" })).StatusCode);
            var list = await other.AdminClient.GetFromJsonAsync<JsonElement>($"/api/v1/screening/matches?customerId={customerId}");
            Assert.Equal(0, list.GetProperty("totalCount").GetInt32());
        }
        finally
        {
            await other.DisposeAsync();
        }
    }

    [Fact]
    public async Task Dashboard_counts_pending_matches()
    {
        await CreateIndividualAsync("Zephaniah", "Mbwana", null);
        var dashboard = await _fx.AdminClient.GetFromJsonAsync<JsonElement>("/api/v1/dashboard");
        Assert.True(dashboard.GetProperty("screening").GetProperty("pendingReview").GetInt32() >= 1);
    }

    private async Task<Guid> CreateIndividualAsync(string first, string last, string? dateOfBirth)
    {
        var response = await _fx.AdminClient.PostAsJsonAsync("/api/v1/customers", new
        {
            type = "INDIVIDUAL", country = "KE", firstName = first, lastName = last, dateOfBirth
        });
        response.EnsureSuccessStatusCode();
        return (await response.Content.ReadFromJsonAsync<JsonElement>()).GetProperty("id").GetGuid();
    }

    private Task<JsonElement> MatchesAsync(Guid customerId)
        => _fx.AdminClient.GetFromJsonAsync<JsonElement>($"/api/v1/screening/matches?customerId={customerId}");

    private async Task<JsonElement> SingleMatchAsync(Guid customerId)
        => Assert.Single((await MatchesAsync(customerId)).GetProperty("items").EnumerateArray());

    private Task<HttpResponseMessage> UploadAsync(string csv)
    {
        var form = new MultipartFormDataContent
        {
            { new ByteArrayContent(Encoding.UTF8.GetBytes(csv)), "file", "list.csv" }
        };
        return _fx.AdminClient.PutAsync("/api/v1/screening/tenant-list", form);
    }
}
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `dotnet test tests/Integration --filter "FullyQualifiedName~ScreeningApiTests"`
Expected: FAIL with 404s (no controller) and compile errors for the missing permissions.

- [ ] **Step 3: Permissions**

In `Permissions.cs` add the constants and include them in `All`:

```csharp
public const string ScreeningRead = "screening.read";
public const string ScreeningReview = "screening.review";
public const string ScreeningManage = "screening.manage";
```

In `RolePermissions`: append `Permissions.ScreeningRead` to `ViewerGrants`, and change `ReviewerGrants` to:

```csharp
private static readonly string[] ReviewerGrants = AnalystGrants
    .Append(Permissions.AuditRead)
    .Append(Permissions.ScreeningReview)
    .ToArray();
```

Admin already gets `Permissions.All`. If `tests/Unit/Security` asserts the exact grants per role, add the new codes there.

- [ ] **Step 4: Customer date of birth and screening on create**

In `CustomersController`:
- Add `string? DateOfBirth` as the last member of `CreateCustomerRequest`.
- Inject `ICustomerScreeningService screening` (store as `_screening`).
- Before creating an individual:

```csharp
DateOnly? dateOfBirth = null;
if (!string.IsNullOrWhiteSpace(request.DateOfBirth))
{
    if (!DateOnly.TryParseExact(request.DateOfBirth, "yyyy-MM-dd", CultureInfo.InvariantCulture, DateTimeStyles.None, out var dob)
        || dob > DateOnly.FromDateTime(DateTime.UtcNow))
        return BadRequest("DateOfBirth must be a past date in yyyy-MM-dd format.");
    dateOfBirth = dob;
}
```

  and pass `dateOfBirth` to `Customer.CreateIndividual(...)`.
- After the existing `_risk.TryRecalculateAsync(...)` call:

```csharp
await _screening.TryScreenCustomerAsync(
    _tenant.TenantId, customer.Id, ScreeningTriggers.CustomerCreated,
    new RiskActor(_tenant.UserId.ToString(), _tenant.Roles.FirstOrDefault(), HttpContext.TraceIdentifier), ct);
```

- Add `dateOfBirth = customer.DateOfBirth` to `ToCustomerResponse`.

Usings: `System.Globalization`, `Aegis.Application.Screening`, `Aegis.Modules.Screening.Domain`.

- [ ] **Step 5: Dashboard pending count**

`DashboardMetrics.cs`: add `public sealed record ScreeningMetrics(int PendingReview);` and `Task<ScreeningMetrics> GetScreeningMetricsAsync(TenantId tenantId, CancellationToken cancellationToken = default);` to `IDashboardQueries`.

`DashboardQueries.cs`:

```csharp
public async Task<ScreeningMetrics> GetScreeningMetricsAsync(TenantId tenantId, CancellationToken cancellationToken = default)
    => new(await _db.ScreeningMatches.AsNoTracking()
        .CountAsync(m => m.TenantId == tenantId && m.Status == ScreeningMatchStatus.PENDING_REVIEW, cancellationToken));
```

`DashboardController.cs`: add `ScreeningMetrics? Screening` as the last member of `DashboardResponse`, and:

```csharp
var screening = Can(Permissions.ScreeningRead) ? await _queries.GetScreeningMetricsAsync(tenantId, ct) : null;
return new DashboardResponse(now, _options.AlertSlaDays, _options.CaseSlaDays, alerts, cases, transactions, risk, screening);
```

If a unit test fakes `IDashboardQueries`, add the method to the fake.

- [ ] **Step 6: Screening controller**

Create `src/Aegis.Api/Controllers/ScreeningController.cs`:

```csharp
namespace Aegis.Api.Controllers;

using Aegis.Api.Authorization;
using Aegis.Application.Risk;
using Aegis.Application.Screening;
using Aegis.Modules.Audit.Application;
using Aegis.Modules.Audit.Domain;
using Aegis.Modules.Screening.Application;
using Aegis.Modules.Screening.Domain;
using Aegis.Shared.Persistence;
using Aegis.Shared.Security;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;

[ApiController]
[Authorize]
[Route("api/v1")]
public sealed class ScreeningController : ControllerBase
{
    private const long MaxListBytes = 5 * 1024 * 1024;

    private readonly ICustomerScreeningService _screening;
    private readonly IScreeningMatchRepository _matches;
    private readonly IWatchlistRepository _lists;
    private readonly ITenantWatchlistRepository _tenantLists;
    private readonly IWatchlistRefresher _refresher;
    private readonly IAuditWriter _audit;
    private readonly IUnitOfWork _uow;
    private readonly ITenantContext _tenant;

    public ScreeningController(
        ICustomerScreeningService screening,
        IScreeningMatchRepository matches,
        IWatchlistRepository lists,
        ITenantWatchlistRepository tenantLists,
        IWatchlistRefresher refresher,
        IAuditWriter audit,
        IUnitOfWork uow,
        ITenantContext tenant)
    {
        _screening = screening;
        _matches = matches;
        _lists = lists;
        _tenantLists = tenantLists;
        _refresher = refresher;
        _audit = audit;
        _uow = uow;
        _tenant = tenant;
    }

    public sealed record ReviewRequest(string? Reason);

    public sealed record MatchSummaryDto(
        Guid Id, Guid CustomerId, string CustomerName, string Source, string Category, string MatchedName,
        double Score, bool BirthYearMatched, string Status, string Trigger, DateTimeOffset CreatedAt, DateTimeOffset? ReviewedAt);

    public sealed record SubjectDto(
        string Source, string ExternalId, string Category, string Kind, IReadOnlyList<string> Names,
        IReadOnlyList<string> BirthYears, IReadOnlyList<string> Countries, string? Detail);

    public sealed record MatchDetailDto(MatchSummaryDto Match, SubjectDto Subject, string? ReviewedBy, string? ReviewReason, Guid? AlertId);

    public sealed record ListStatusDto(string Source, DateTimeOffset LoadedAt, DateTimeOffset? PublishedAt, int EntryCount);

    public sealed record ListsResponse(IReadOnlyList<ListStatusDto> Sources, TenantWatchlistSummary TenantList);

    public sealed record RefreshResultDto(string Source, string Status, int EntryCount, string? Error);

    public sealed record RefreshResponse(IReadOnlyList<RefreshResultDto> Results, int NewMatches);

    public sealed record UploadResponse(int EntryCount, int NewMatches);

    public sealed record ScreenResponse(int NewMatches, IReadOnlyList<Guid> MatchIds);

    [HttpGet("screening/lists")]
    [RequirePermission(Permissions.ScreeningRead)]
    public async Task<ActionResult<ListsResponse>> Lists(CancellationToken ct)
    {
        if (!_tenant.IsAuthenticated) return Unauthorized();
        var loads = await _lists.ListLatestLoadsAsync(ct);
        return new ListsResponse(
            loads.Select(l => new ListStatusDto(l.Source.ToString(), l.LoadedAt, l.PublishedAt, l.EntryCount)).ToList(),
            await _tenantLists.GetSummaryAsync(_tenant.TenantId, ct));
    }

    /// <summary>Refreshes the shared lists now and rescreens this tenant; other tenants are rescreened by the daily job.</summary>
    [HttpPost("screening/lists/refresh")]
    [RequirePermission(Permissions.ScreeningManage)]
    public async Task<ActionResult<RefreshResponse>> Refresh(CancellationToken ct)
    {
        if (!_tenant.IsAuthenticated) return Unauthorized();
        var results = new List<WatchlistRefreshResult>();
        foreach (var source in _refresher.Sources) results.Add(await _refresher.RefreshAsync(source, ct));

        var newMatches = 0;
        if (results.Any(r => r.Status == WatchlistRefreshStatus.LOADED))
            newMatches = (await _screening.ScreenTenantAsync(_tenant.TenantId, ScreeningTriggers.ListUpdated, Actor(), ct)).NewMatchIds.Count;

        return new RefreshResponse(
            results.Select(r => new RefreshResultDto(r.Source.ToString(), r.Status.ToString(), r.EntryCount, r.Error)).ToList(),
            newMatches);
    }

    [HttpPut("screening/tenant-list")]
    [RequirePermission(Permissions.ScreeningManage)]
    [RequestSizeLimit(MaxListBytes + 64 * 1024)]
    public async Task<IActionResult> UploadTenantList(IFormFile? file, CancellationToken ct)
    {
        if (!_tenant.IsAuthenticated) return Unauthorized();
        if (file is null || file.Length == 0) return BadRequest(new { errors = new[] { "Attach a CSV file as 'file'." } });
        if (file.Length > MaxListBytes) return BadRequest(new { errors = new[] { "The file is larger than 5 MB." } });

        string csv;
        using (var reader = new StreamReader(file.OpenReadStream()))
            csv = await reader.ReadToEndAsync(ct);

        var parsed = TenantWatchlistCsvParser.Parse(csv);
        if (parsed.Errors.Count > 0) return BadRequest(new { errors = parsed.Errors });

        var uploadedBy = _tenant.UserId.ToString();
        await _tenantLists.ReplaceAsync(
            _tenant.TenantId,
            parsed.Subjects.Select(s => TenantWatchlistEntry.From(_tenant.TenantId, s, uploadedBy)).ToList(),
            ct);
        await _audit.AppendAsync(AuditEvent.Create(
            _tenant.TenantId.Value, AuditEventTypes.SCREENING_LIST_UPLOADED, "TenantWatchlist", _tenant.TenantId.Value.ToString(),
            uploadedBy, _tenant.Roles.FirstOrDefault(), null,
            AuditPayload.Json(new { entryCount = parsed.Subjects.Count, fileName = file.FileName }),
            $"Screening list replaced with {parsed.Subjects.Count} entries", HttpContext.TraceIdentifier), ct);
        await _uow.SaveChangesAsync(ct);

        var outcome = await _screening.ScreenTenantAsync(_tenant.TenantId, ScreeningTriggers.ListUpdated, Actor(), ct);
        return Ok(new UploadResponse(parsed.Subjects.Count, outcome.NewMatchIds.Count));
    }

    [HttpGet("screening/matches")]
    [RequirePermission(Permissions.ScreeningRead)]
    public async Task<IActionResult> ListMatches(
        [FromQuery] string? status, [FromQuery] Guid? customerId,
        [FromQuery] int page = 1, [FromQuery] int pageSize = 50, CancellationToken ct = default)
    {
        if (!_tenant.IsAuthenticated) return Unauthorized();
        ScreeningMatchStatus? parsedStatus = null;
        if (!string.IsNullOrWhiteSpace(status))
        {
            if (!Enum.TryParse<ScreeningMatchStatus>(status, true, out var s) || !Enum.IsDefined(s))
                return BadRequest("Status must be PENDING_REVIEW, CONFIRMED or FALSE_POSITIVE.");
            parsedStatus = s;
        }

        var result = await _matches.ListAsync(_tenant.TenantId, new ScreeningMatchQuery(parsedStatus, customerId, page, pageSize), ct);
        return Ok(new
        {
            items = result.Items.Select(ToSummary).ToList(),
            page = result.Page,
            pageSize = result.PageSize,
            totalCount = result.TotalCount
        });
    }

    [HttpGet("screening/matches/{id:guid}")]
    [RequirePermission(Permissions.ScreeningRead)]
    public async Task<ActionResult<MatchDetailDto>> GetMatch(Guid id, CancellationToken ct)
    {
        if (!_tenant.IsAuthenticated) return Unauthorized();
        var match = await _matches.GetAsync(_tenant.TenantId, id, ct);
        return match is null ? NotFound() : ToDetail(match);
    }

    [HttpPost("screening/matches/{id:guid}/confirm")]
    [RequirePermission(Permissions.ScreeningReview)]
    public Task<IActionResult> Confirm(Guid id, [FromBody] ReviewRequest request, CancellationToken ct)
        => ReviewAsync(id, ScreeningDecision.CONFIRM, request, ct);

    [HttpPost("screening/matches/{id:guid}/dismiss")]
    [RequirePermission(Permissions.ScreeningReview)]
    public Task<IActionResult> Dismiss(Guid id, [FromBody] ReviewRequest request, CancellationToken ct)
        => ReviewAsync(id, ScreeningDecision.FALSE_POSITIVE, request, ct);

    [HttpPost("customers/{id:guid}/screen")]
    [RequirePermission(Permissions.CustomerWrite)]
    public async Task<ActionResult<ScreenResponse>> ScreenCustomer(Guid id, CancellationToken ct)
    {
        if (!_tenant.IsAuthenticated) return Unauthorized();
        var outcome = await _screening.ScreenCustomerAsync(_tenant.TenantId, id, ScreeningTriggers.Manual, Actor(), ct);
        return outcome is null ? NotFound() : new ScreenResponse(outcome.NewMatchIds.Count, outcome.NewMatchIds);
    }

    private async Task<IActionResult> ReviewAsync(Guid id, ScreeningDecision decision, ReviewRequest request, CancellationToken ct)
    {
        if (!_tenant.IsAuthenticated) return Unauthorized();
        if (string.IsNullOrWhiteSpace(request.Reason)) return BadRequest("A reason is required.");
        try
        {
            var match = await _screening.ReviewAsync(_tenant.TenantId, id, decision, request.Reason, Actor(), ct);
            return match is null ? NotFound() : Ok(ToSummary(match));
        }
        catch (InvalidOperationException ex)
        {
            return Conflict(ex.Message);
        }
        catch (UniqueConstraintViolationException)
        {
            return Conflict("This match was reviewed by someone else. Reload to see the decision.");
        }
    }

    private RiskActor Actor() => new(_tenant.UserId.ToString(), _tenant.Roles.FirstOrDefault(), HttpContext.TraceIdentifier);

    private static MatchSummaryDto ToSummary(ScreeningMatch m) => new(
        m.Id, m.CustomerId, m.CustomerName, m.Source.ToString(), m.Category.ToString(), m.MatchedName,
        m.Score, m.BirthYearMatched, m.Status.ToString(), m.Trigger, m.CreatedAt, m.ReviewedAt);

    private static MatchDetailDto ToDetail(ScreeningMatch m) => new(
        ToSummary(m),
        new SubjectDto(
            m.Subject.Source.ToString(), m.Subject.ExternalId, m.Subject.Category.ToString(), m.Subject.Kind.ToString(),
            m.Subject.Names,
            m.Subject.BirthYears.Select(r => r.From == r.To ? $"{r.From}" : $"{r.From}–{r.To}").ToList(),
            m.Subject.Countries, m.Subject.Detail),
        m.ReviewedBy, m.ReviewReason, m.AlertId);
}
```

- [ ] **Step 7: Run the tests to verify they pass**

Run: `dotnet test tests/Unit && dotnet test tests/Integration`
Expected: all pass, including `ScreeningApiTests`.

- [ ] **Step 8: Commit**

```bash
git add src tests
git commit -m "feat(screening): screening API, permissions, screen on customer create, dashboard count"
```

---

### Task 8: Risk factors for confirmed matches

**Files:**
- Modify: `src/Aegis.Modules.Risk/Domain/RiskTypes.cs`, `RiskCalculator.cs`, `RiskModel.cs` (defaults)
- Modify: `src/Aegis.Infrastructure/Persistence/Repositories/RiskRepositories.cs` (`RiskInputsReader`)
- Create: `src/Aegis.Infrastructure/Screening/ScreeningRiskFactorSync.cs`
- Modify: `src/Aegis.Infrastructure/DependencyInjection.cs`
- Modify: `tests/Integration/Risk/CustomerRiskApiTests.cs` (default factor count 6 → 9)
- Test: `tests/Unit/Risk/RiskCalculatorTests.cs`, `tests/Unit/Risk/RiskModelValidationTests.cs`, `tests/Integration/Screening/ScreeningRiskTests.cs`

**Interfaces:**
- Consumes: Task 4 `ScreeningMatch`; Task 7 endpoints.
- Produces:
  - `RiskFactorType.SANCTIONS_MATCH`, `PEP_MATCH`, `WATCHLIST_MATCH` (no parameters; full weight when the customer has a confirmed match of that category)
  - `RiskInputs(..., DateTimeOffset AsOf, IReadOnlyCollection<string>? ConfirmedScreeningCategories = null)`
  - Default model adds `SANCTIONS_MATCH` 100, `WATCHLIST_MATCH` 50, `PEP_MATCH` 30
  - `ScreeningRiskFactorSync.SyncAsync(CancellationToken)` returns the number of tenants updated; runs at startup unless `Risk:SyncScreeningFactorsOnStartup` is `false`

- [ ] **Step 1: Write the failing unit tests**

Add to `tests/Unit/Risk/RiskCalculatorTests.cs` (reuse the file's `Inputs(...)` helper for the other fields):

```csharp
[Theory]
[InlineData("SANCTIONS", RiskFactorType.SANCTIONS_MATCH, 100)]
[InlineData("WATCHLIST", RiskFactorType.WATCHLIST_MATCH, 50)]
[InlineData("PEP", RiskFactorType.PEP_MATCH, 30)]
public void Confirmed_screening_match_adds_its_factor_weight(string category, RiskFactorType type, int points)
{
    var inputs = Inputs() with { ConfirmedScreeningCategories = new[] { category } };

    var result = RiskCalculator.Calculate(RiskModelDefaults.Factors, RiskModelDefaults.Bands, inputs);

    Assert.Equal(points, result.Contributions.Single(c => c.Type == type).Points);
}

[Fact]
public void Confirmed_sanctions_match_puts_the_customer_in_the_critical_band()
{
    var inputs = Inputs() with { ConfirmedScreeningCategories = new[] { "SANCTIONS" } };

    var result = RiskCalculator.Calculate(RiskModelDefaults.Factors, RiskModelDefaults.Bands, inputs);

    Assert.Equal(100, result.Score);
    Assert.Equal(RiskBand.CRITICAL, result.Band);
}

[Fact]
public void No_confirmed_match_adds_nothing()
    => Assert.All(
        RiskCalculator.Calculate(RiskModelDefaults.Factors, RiskModelDefaults.Bands, Inputs()).Contributions
            .Where(c => c.Type is RiskFactorType.SANCTIONS_MATCH or RiskFactorType.PEP_MATCH or RiskFactorType.WATCHLIST_MATCH),
        c => Assert.Equal(0, c.Points));
```

Add to `tests/Unit/Risk/RiskModelValidationTests.cs`:

```csharp
[Fact]
public void Screening_factors_need_only_a_weight()
    => Assert.Empty(RiskModelValidator.Validate(
        new[]
        {
            new RiskFactorDefinition(RiskFactorType.SANCTIONS_MATCH, 100),
            new RiskFactorDefinition(RiskFactorType.PEP_MATCH, 30),
            new RiskFactorDefinition(RiskFactorType.WATCHLIST_MATCH, 50)
        },
        RiskModelDefaults.Bands));
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `dotnet test tests/Unit --filter "FullyQualifiedName~Risk"`
Expected: build fails (unknown enum members, no `ConfirmedScreeningCategories`).

- [ ] **Step 3: Implement the factors**

`RiskTypes.cs`: add `SANCTIONS_MATCH, PEP_MATCH, WATCHLIST_MATCH` to `RiskFactorType`, and append the parameter to `RiskInputs`:

```csharp
public sealed record RiskInputs(
    string CustomerType,
    string Country,
    IReadOnlyList<RiskAlertFact> Alerts,
    int SuspiciousCaseCount,
    IReadOnlyDictionary<int, int> TransactionCounts,
    DateTimeOffset AsOf,
    IReadOnlyCollection<string>? ConfirmedScreeningCategories = null);
```

`RiskCalculator.cs`: add to the switch

```csharp
RiskFactorType.SANCTIONS_MATCH => Screening(f, inputs, "SANCTIONS", "sanctions"),
RiskFactorType.PEP_MATCH => Screening(f, inputs, "PEP", "PEP"),
RiskFactorType.WATCHLIST_MATCH => Screening(f, inputs, "WATCHLIST", "watchlist"),
```

and the helper

```csharp
private static RiskContribution Screening(RiskFactorDefinition f, RiskInputs inputs, string category, string label)
{
    var hit = inputs.ConfirmedScreeningCategories?.Contains(category, StringComparer.OrdinalIgnoreCase) == true;
    return new RiskContribution(f.Type, hit ? f.Weight : 0, f.Weight, hit ? $"confirmed {label} match" : $"no confirmed {label} match");
}
```

`RiskModel.cs`: append to `RiskModelDefaults.Factors`

```csharp
new RiskFactorDefinition(RiskFactorType.SANCTIONS_MATCH, 100),
new RiskFactorDefinition(RiskFactorType.WATCHLIST_MATCH, 50),
new RiskFactorDefinition(RiskFactorType.PEP_MATCH, 30),
```

`RiskModelValidator` needs no change: the new types fall through the switch with only the weight check.

`RiskInputsReader.ReadAsync` (`RiskRepositories.cs`): before the `return`, add

```csharp
var confirmed = await _db.ScreeningMatches.AsNoTracking()
    .Where(m => m.TenantId == tenantId && m.CustomerId == customerId && m.Status == ScreeningMatchStatus.CONFIRMED)
    .Select(m => m.Category)
    .Distinct()
    .ToListAsync(cancellationToken);
```

and pass `confirmed.Select(c => c.ToString()).ToList()` as the last `RiskInputs` argument (`using Aegis.Modules.Screening.Domain;`).

- [ ] **Step 4: Add the factors to existing tenants' models once**

Create `src/Aegis.Infrastructure/Screening/ScreeningRiskFactorSync.cs`:

```csharp
namespace Aegis.Infrastructure.Screening;

using Aegis.Application.Risk;
using Aegis.Infrastructure.Persistence;
using Aegis.Modules.Risk.Domain;
using Aegis.Shared.Domain;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

/// <summary>
/// Gives tenants whose risk model predates screening the three screening factors, as a new audited version.
/// A tenant that ever had one of them is skipped, so an admin who later removes a factor keeps that choice.
/// </summary>
public sealed class ScreeningRiskFactorSync
{
    private static readonly RiskFactorType[] Types =
        { RiskFactorType.SANCTIONS_MATCH, RiskFactorType.PEP_MATCH, RiskFactorType.WATCHLIST_MATCH };

    private readonly AegisDbContext _db;
    private readonly ICustomerRiskService _risk;

    public ScreeningRiskFactorSync(AegisDbContext db, ICustomerRiskService risk)
    {
        _db = db;
        _risk = risk;
    }

    public async Task<int> SyncAsync(TenantId? onlyTenant = null, CancellationToken cancellationToken = default)
    {
        var query = _db.RiskModels.AsNoTracking();
        if (onlyTenant is { } tenant) query = query.Where(m => m.TenantId == tenant);
        var models = await query.ToListAsync(cancellationToken);
        var alreadyHad = models.Where(m => m.Factors.Any(f => Types.Contains(f.Type))).Select(m => m.TenantId).ToHashSet();
        var defaults = RiskModelDefaults.Factors.Where(f => Types.Contains(f.Type)).ToList();
        var actor = new RiskActor("system-seed", null, null);

        var updated = 0;
        foreach (var active in models.Where(m => m.IsActive && !alreadyHad.Contains(m.TenantId)))
        {
            await _risk.UpdateModelAsync(active.TenantId, active.Factors.Concat(defaults).ToList(), active.Bands, actor, cancellationToken);
            updated++;
        }
        return updated;
    }
}

public sealed class ScreeningRiskFactorSyncService : IHostedService
{
    private readonly IServiceScopeFactory _scopes;
    private readonly IConfiguration _config;

    public ScreeningRiskFactorSyncService(IServiceScopeFactory scopes, IConfiguration config)
    {
        _scopes = scopes;
        _config = config;
    }

    public async Task StartAsync(CancellationToken cancellationToken)
    {
        if (!_config.GetValue("Risk:SyncScreeningFactorsOnStartup", true)) return;
        await using var scope = _scopes.CreateAsyncScope();
        await scope.ServiceProvider.GetRequiredService<ScreeningRiskFactorSync>().SyncAsync(null, cancellationToken);
    }

    public Task StopAsync(CancellationToken cancellationToken) => Task.CompletedTask;
}
```

Register:

```csharp
services.AddScoped<Screening.ScreeningRiskFactorSync>();
services.AddHostedService<Screening.ScreeningRiskFactorSyncService>();
```

- [ ] **Step 5: Integration tests**

In `tests/Integration/Risk/CustomerRiskApiTests.cs`, change `Assert.Equal(6, current.GetProperty("contributions").GetArrayLength());` to `9`.

Create `tests/Integration/Screening/ScreeningRiskTests.cs`:

```csharp
namespace Aegis.Tests.Integration.Screening;

using System.Net.Http.Json;
using System.Text.Json;
using Aegis.Infrastructure.Screening;
using Aegis.Tests.Integration.Alerts;
using Microsoft.Extensions.DependencyInjection;

public sealed class ScreeningRiskTests : IAsyncLifetime
{
    private readonly AlertTestFixture _fx = new();

    public async Task InitializeAsync()
    {
        await _fx.InitializeAsync();
        (await _fx.AdminClient.PostAsync("/api/v1/screening/lists/refresh", null)).EnsureSuccessStatusCode();
    }

    public Task DisposeAsync() => _fx.DisposeAsync();

    [Fact]
    public async Task Confirmed_sanctions_match_makes_the_customer_critical()
    {
        var created = await _fx.AdminClient.PostAsJsonAsync("/api/v1/customers", new
        {
            type = "INDIVIDUAL", country = "KE", firstName = "Zephaniah", lastName = "Mbwana"
        });
        var customerId = (await created.Content.ReadFromJsonAsync<JsonElement>()).GetProperty("id").GetGuid();
        var matches = await _fx.AdminClient.GetFromJsonAsync<JsonElement>($"/api/v1/screening/matches?customerId={customerId}");
        var matchId = matches.GetProperty("items")[0].GetProperty("id").GetGuid();

        (await _fx.AdminClient.PostAsJsonAsync($"/api/v1/screening/matches/{matchId}/confirm", new { reason = "Same ID" }))
            .EnsureSuccessStatusCode();

        var current = (await _fx.AdminClient.GetFromJsonAsync<JsonElement>($"/api/v1/customers/{customerId}/risk")).GetProperty("current");
        Assert.Equal("CRITICAL", current.GetProperty("band").GetString());
        Assert.Equal("SCREENING_CONFIRMED", current.GetProperty("trigger").GetString());
        var sanctions = current.GetProperty("contributions").EnumerateArray()
            .Single(c => c.GetProperty("type").GetString() == "SANCTIONS_MATCH");
        Assert.Equal(100, sanctions.GetProperty("points").GetInt32());
    }

    [Fact]
    public async Task Sync_adds_screening_factors_to_a_model_that_predates_them_once()
    {
        var tenant = await StripScreeningFactorsFromAllVersionsAsync();

        Assert.Equal(1, await SyncAsync(tenant));
        Assert.Contains("SANCTIONS_MATCH", await ActiveFactorTypesAsync());
        Assert.Equal(0, await SyncAsync(tenant));
    }

    [Fact]
    public async Task Sync_keeps_an_admins_removal_of_screening_factors()
    {
        var model = await _fx.AdminClient.GetFromJsonAsync<JsonElement>("/api/v1/risk/model");
        var withoutScreening = model.GetProperty("factors").EnumerateArray()
            .Where(f => !f.GetProperty("type").GetString()!.EndsWith("_MATCH"))
            .Select(f => JsonSerializer.Deserialize<object>(f.GetRawText()))
            .ToList();
        (await _fx.AdminClient.PutAsJsonAsync("/api/v1/risk/model", new { factors = withoutScreening, bands = model.GetProperty("bands") }))
            .EnsureSuccessStatusCode();

        Assert.Equal(0, await SyncAsync(await TenantIdAsync()));
        Assert.DoesNotContain("SANCTIONS_MATCH", await ActiveFactorTypesAsync());
    }

    private async Task<TenantId> TenantIdAsync()
    {
        using var scope = _fx.Factory.Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AegisDbContext>();
        var id = await db.Tenants.IgnoreQueryFilters().Where(t => t.Slug == _fx.Slug).Select(t => t.Id).SingleAsync();
        return new TenantId(id);
    }

    private async Task<TenantId> StripScreeningFactorsFromAllVersionsAsync()
    {
        await _fx.AdminClient.GetAsync("/api/v1/risk/model");
        var tenant = await TenantIdAsync();
        using var scope = _fx.Factory.Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AegisDbContext>();
        await db.Database.ExecuteSqlInterpolatedAsync($"""
            UPDATE risk.risk_models
            SET factors_json = (
                SELECT jsonb_agg(f) FROM jsonb_array_elements(factors_json) f
                WHERE f->>'Type' NOT IN ('SANCTIONS_MATCH', 'PEP_MATCH', 'WATCHLIST_MATCH'))
            WHERE tenant_id = {tenant.Value}
            """);
        return tenant;
    }

    private async Task<int> SyncAsync(TenantId tenant)
    {
        using var scope = _fx.Factory.Services.CreateScope();
        return await scope.ServiceProvider.GetRequiredService<ScreeningRiskFactorSync>().SyncAsync(tenant);
    }

    private async Task<List<string?>> ActiveFactorTypesAsync()
    {
        var model = await _fx.AdminClient.GetFromJsonAsync<JsonElement>("/api/v1/risk/model");
        return model.GetProperty("factors").EnumerateArray().Select(f => f.GetProperty("type").GetString()).ToList();
    }
}
```

Add `using Aegis.Infrastructure.Persistence;`, `using Aegis.Shared.Domain;` and `using Microsoft.EntityFrameworkCore;` to the test file. Before writing the helpers, check three names against the code: the tenant entity's `Slug` and `Id` properties, the `TenantId` constructor, and the stored factor JSON key. `RiskJson` has no naming policy, so the key should be PascalCase `Type`. The first test covers the upgrade path. The second covers the "keep the admin's choice" rule: this tenant's version 1 came from the new defaults, so it already had the screening factors, and after the admin removes them the sync leaves the model alone.

- [ ] **Step 6: Run the tests**

Run: `dotnet test tests/Unit && dotnet test tests/Integration`
Expected: all pass.

- [ ] **Step 7: Commit**

```bash
git add src tests
git commit -m "feat(risk): sanctions, PEP and watchlist match factors; add them to existing models once"
```

---

### Task 9: Console — permissions, navigation, API client, helpers, alert headline

**Files (in `aegis-console`):**
- Modify: `src/lib/auth/permissions.ts` (+ `permissions.test.ts` if it lists grants)
- Modify: `src/lib/nav.ts` (+ `nav.test.ts`)
- Modify: `src/components/shell/AppShell.tsx`, `src/lib/api/dashboard.ts`
- Create: `src/lib/screening/match.ts`, `src/lib/screening/match.test.ts`
- Create: `src/lib/api/screening.ts`
- Modify: `src/lib/api/types.ts` (`Customer.dateOfBirth`)
- Modify: `src/lib/alerts/headline.ts`, `src/lib/alerts/headline.test.ts`
- Modify: `src/lib/risk/model.ts` (factor types, labels, params)

**Interfaces:**
- Consumes: the Task 7 API.
- Produces:
  - `Permissions.ScreeningRead | ScreeningReview | ScreeningManage`
  - `NavBadge` adds `"pendingScreening"`
  - Types `ScreeningStatus`, `ScreeningCategory`, `ScreeningSource`, `ScreeningMatchSummary`, `ScreeningSubject`, `ScreeningMatchDetail`, `ScreeningLists`; constants `STATUS_LABELS`, `SOURCE_LABELS`, `CATEGORY_LABELS`, `TENANT_LIST_TEMPLATE`; functions `scorePercent(score)`, `isScreeningAlert(additionalContext)`
  - API functions `listScreeningMatches`, `getScreeningMatch`, `confirmScreeningMatch`, `dismissScreeningMatch`, `getScreeningLists`, `refreshScreeningLists`, `uploadTenantList`, `screenCustomer`

- [ ] **Step 1: Write the failing tests**

Create `src/lib/screening/match.test.ts`:

```ts
import assert from "node:assert/strict";
import { test } from "node:test";
import { CATEGORY_LABELS, isScreeningAlert, scorePercent, SOURCE_LABELS, STATUS_LABELS, TENANT_LIST_TEMPLATE } from "./match.ts";

test("scorePercent rounds to whole percent", () => {
  assert.equal(scorePercent(0.9333), "93%");
  assert.equal(scorePercent(1), "100%");
});

test("every status, source and category has a label", () => {
  assert.deepEqual(Object.keys(STATUS_LABELS).sort(), ["CONFIRMED", "FALSE_POSITIVE", "PENDING_REVIEW"]);
  assert.deepEqual(Object.keys(SOURCE_LABELS).sort(), ["OFAC", "TENANT", "UN"]);
  assert.deepEqual(Object.keys(CATEGORY_LABELS).sort(), ["PEP", "SANCTIONS", "WATCHLIST"]);
});

test("template header matches the API's CSV columns", () => {
  assert.equal(TENANT_LIST_TEMPLATE.split("\n")[0], "reference,name,category,kind,aliases,birthYear,country,notes");
});

test("isScreeningAlert reads the rule code", () => {
  assert.equal(isScreeningAlert({ ruleCode: "SCREENING_MATCH" }), true);
  assert.equal(isScreeningAlert({ ruleCode: "STRUCTURING_001" }), false);
  assert.equal(isScreeningAlert(null), false);
});
```

Add to `src/lib/alerts/headline.test.ts`:

```ts
test("screening alert headline names the list and the matched name", () => {
  assert.equal(
    alertHeadline({ list: "UN", category: "SANCTIONS", matchedName: "ZEPHANIAH OKOTH MBWANA", matchScore: 0.9333 }),
    "Confirmed match on the UN Security Council list: ZEPHANIAH OKOTH MBWANA (93% name similarity)",
  );
});
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `npm test`
Expected: FAIL, `./match.ts` not found and the headline test fails.

- [ ] **Step 3: Implement helpers, API client and headline**

Create `src/lib/screening/match.ts`:

```ts
export type ScreeningStatus = "PENDING_REVIEW" | "CONFIRMED" | "FALSE_POSITIVE";
export type ScreeningCategory = "SANCTIONS" | "PEP" | "WATCHLIST";
export type ScreeningSource = "UN" | "OFAC" | "TENANT";

export type ScreeningMatchSummary = {
  id: string;
  customerId: string;
  customerName: string;
  source: ScreeningSource;
  category: ScreeningCategory;
  matchedName: string;
  score: number;
  birthYearMatched: boolean;
  status: ScreeningStatus;
  trigger: string;
  createdAt: string;
  reviewedAt: string | null;
};

export type ScreeningSubject = {
  source: ScreeningSource;
  externalId: string;
  category: ScreeningCategory;
  kind: "INDIVIDUAL" | "ENTITY";
  names: string[];
  birthYears: string[];
  countries: string[];
  detail: string | null;
};

export type ScreeningMatchDetail = {
  match: ScreeningMatchSummary;
  subject: ScreeningSubject;
  reviewedBy: string | null;
  reviewReason: string | null;
  alertId: string | null;
};

export type ScreeningLists = {
  sources: { source: ScreeningSource; loadedAt: string; publishedAt: string | null; entryCount: number }[];
  tenantList: { entryCount: number; uploadedAt: string | null; uploadedBy: string | null };
};

export const STATUS_LABELS: Record<ScreeningStatus, string> = {
  PENDING_REVIEW: "Needs review",
  CONFIRMED: "Confirmed",
  FALSE_POSITIVE: "False positive",
};

export const SOURCE_LABELS: Record<ScreeningSource, string> = {
  UN: "UN Security Council",
  OFAC: "US OFAC SDN",
  TENANT: "Your list",
};

export const CATEGORY_LABELS: Record<ScreeningCategory, string> = {
  SANCTIONS: "Sanctions",
  PEP: "Politically exposed person",
  WATCHLIST: "Internal watchlist",
};

/** Mirrors TenantWatchlistCsvParser on the API. */
export const TENANT_LIST_TEMPLATE =
  "reference,name,category,kind,aliases,birthYear,country,notes\n" +
  "PEP-001,Jane Wanjiku Doe,PEP,INDIVIDUAL,Jane W. Doe;J. Wanjiku,1968,KE,Former county governor\n" +
  "INT-001,Blue Lagoon Traders,WATCHLIST,ENTITY,,,KE,Exited relationship 2025\n";

export function scorePercent(score: number): string {
  return `${Math.round(score * 100)}%`;
}

export function isScreeningAlert(additionalContext: Record<string, unknown> | null | undefined): boolean {
  return additionalContext?.ruleCode === "SCREENING_MATCH";
}
```

Create `src/lib/api/screening.ts`:

```ts
import { apiFetch } from "@/lib/api/client";
import type { ScreeningLists, ScreeningMatchDetail, ScreeningMatchSummary, ScreeningStatus } from "@/lib/screening/match";

export type ScreeningMatchPage = { items: ScreeningMatchSummary[]; page: number; pageSize: number; totalCount: number };

export function listScreeningMatches(params: {
  status?: ScreeningStatus;
  customerId?: string;
  page?: number;
  pageSize?: number;
}): Promise<ScreeningMatchPage> {
  const search = new URLSearchParams();
  if (params.status) search.set("status", params.status);
  if (params.customerId) search.set("customerId", params.customerId);
  search.set("page", String(params.page ?? 1));
  search.set("pageSize", String(params.pageSize ?? 50));
  return apiFetch<ScreeningMatchPage>(`/api/v1/screening/matches?${search}`);
}

export function getScreeningMatch(id: string): Promise<ScreeningMatchDetail> {
  return apiFetch<ScreeningMatchDetail>(`/api/v1/screening/matches/${id}`);
}

export function confirmScreeningMatch(id: string, reason: string): Promise<ScreeningMatchSummary> {
  return apiFetch<ScreeningMatchSummary>(`/api/v1/screening/matches/${id}/confirm`, { method: "POST", body: { reason } });
}

export function dismissScreeningMatch(id: string, reason: string): Promise<ScreeningMatchSummary> {
  return apiFetch<ScreeningMatchSummary>(`/api/v1/screening/matches/${id}/dismiss`, { method: "POST", body: { reason } });
}

export function getScreeningLists(): Promise<ScreeningLists> {
  return apiFetch<ScreeningLists>("/api/v1/screening/lists");
}

export type RefreshResult = { results: { source: string; status: string; entryCount: number; error: string | null }[]; newMatches: number };

export function refreshScreeningLists(): Promise<RefreshResult> {
  return apiFetch<RefreshResult>("/api/v1/screening/lists/refresh", { method: "POST" });
}

export function uploadTenantList(file: File): Promise<{ entryCount: number; newMatches: number }> {
  const form = new FormData();
  form.append("file", file);
  return apiFetch<{ entryCount: number; newMatches: number }>("/api/v1/screening/tenant-list", { method: "PUT", body: form });
}

export function screenCustomer(customerId: string): Promise<{ newMatches: number; matchIds: string[] }> {
  return apiFetch<{ newMatches: number; matchIds: string[] }>(`/api/v1/customers/${customerId}/screen`, { method: "POST" });
}
```

The tenant-list upload returns `400 { errors: [...] }`. Check how `ApiError` and `errorMessage` surface a JSON body (`src/lib/api/client.ts`, `errorText.ts`). If they only show plain text, extend `errorText` to join an `errors` array with line breaks, with a test in `errorText.test.ts`.

In `headline.ts`, at the top of `alertHeadline`, before any other logic:

```ts
if (typeof evaluatedValues?.matchedName === "string") {
  const list = SOURCE_LABELS[evaluatedValues.list as ScreeningSource] ?? String(evaluatedValues.list ?? "a screening");
  const score = num(evaluatedValues, "matchScore");
  const similarity = score === null ? "" : ` (${scorePercent(score)} name similarity)`;
  const listPhrase = evaluatedValues.list === "TENANT" ? "your screening list" : `the ${list} list`;
  return `Confirmed match on ${listPhrase}: ${evaluatedValues.matchedName}${similarity}`;
}
```

with `import { SOURCE_LABELS, scorePercent, type ScreeningSource } from "../screening/match.ts";`.

- [ ] **Step 4: Permissions, navigation, badge, risk labels, customer type**

`permissions.ts`: add `ScreeningRead: "screening.read"`, `ScreeningReview: "screening.review"`, `ScreeningManage: "screening.manage"`; add `P.ScreeningRead` to `VIEWER`; change `REVIEWER` to `[...ANALYST, P.AuditRead, P.ScreeningReview]`.

`nav.ts`: `export type NavBadge = "openAlerts" | "openCases" | "pendingScreening";`. In "Work", after Cases: `{ href: "/screening", label: "Screening", permission: Permissions.ScreeningRead, badge: "pendingScreening" }`. In "Admin": `{ href: "/admin/screening", label: "Screening lists", permission: Permissions.ScreeningManage }`. Update `nav.test.ts` expectations that list links per role.

`dashboard.ts`: add `export type ScreeningMetrics = { pendingReview: number };` and `screening: ScreeningMetrics | null;` to `Dashboard`.

`AppShell.tsx`: add `pendingScreening: dashboard.data?.screening?.pendingReview,` to `badges`. Check `Sidebar`'s `badges` prop type and widen it if it is keyed by `NavBadge`.

`types.ts`: add `dateOfBirth: string | null;` to the customer type.

`risk/model.ts`: add `"SANCTIONS_MATCH" | "PEP_MATCH" | "WATCHLIST_MATCH"` to `RiskFactorType`, append them to `FACTOR_TYPES`, add labels (`"Confirmed sanctions match"`, `"Confirmed PEP match"`, `"Confirmed watchlist match"`) and `[]` in `FACTOR_PARAMS`. Update `model.test.ts` if it counts factor types.

- [ ] **Step 5: Run checks**

Run: `npm test && npm run lint && npx tsc --noEmit`
Expected: all pass.

- [ ] **Step 6: Commit**

```bash
git add src
git commit -m "feat(screening): console permissions, navigation, API client and screening alert headline"
```

---

### Task 10: Console — queue, match review, customer panel, list admin

**Files (in `aegis-console`):**
- Create: `src/app/(app)/screening/page.tsx`, `src/app/(app)/screening/[id]/page.tsx`, `src/app/(app)/admin/screening/page.tsx`
- Create: `src/components/screening/ScreeningPanel.tsx`
- Modify: `src/app/(app)/customers/[id]/page.tsx` (panel), `src/app/(app)/alerts/[id]/page.tsx` (link to the match)

**Interfaces:**
- Consumes: Task 9 helpers and API functions; `components/ui` (`PageHeader`, `DataTable`, `Tabs`, `Pagination`, `Field`, `Button`, `inputClass`, `Stat`, `States`); `useApiQuery`, `errorMessage`; `useSession().can`; `formatDateTime`.

- [ ] **Step 1: Review queue**

Create `src/app/(app)/screening/page.tsx`:

```tsx
"use client";

import Link from "next/link";
import { useSearchParams } from "next/navigation";
import { Suspense } from "react";
import { DataTable } from "@/components/ui/DataTable";
import { PageHeader } from "@/components/ui/PageHeader";
import { Pagination } from "@/components/ui/Pagination";
import { EmptyState, ErrorState, LoadingState } from "@/components/ui/States";
import { Tabs } from "@/components/ui/Tabs";
import { listScreeningMatches } from "@/lib/api/screening";
import { formatDateTime } from "@/lib/format/time";
import { useApiQuery } from "@/lib/hooks/useApiQuery";
import {
  CATEGORY_LABELS,
  scorePercent,
  SOURCE_LABELS,
  STATUS_LABELS,
  type ScreeningMatchSummary,
  type ScreeningStatus,
} from "@/lib/screening/match";

const PAGE_SIZE = 50;
const VIEWS: ScreeningStatus[] = ["PENDING_REVIEW", "CONFIRMED", "FALSE_POSITIVE"];
const linkClass = "text-[var(--accent-strong)] underline-offset-2 hover:underline";

export default function ScreeningPage() {
  return (
    <Suspense fallback={<LoadingState />}>
      <ScreeningQueue />
    </Suspense>
  );
}

function ScreeningQueue() {
  const params = useSearchParams();
  const status = VIEWS.find((v) => v === params.get("status")) ?? "PENDING_REVIEW";
  const page = Math.max(1, Number(params.get("page")) || 1);
  const query = useApiQuery(`screening:${status}:${page}`, () => listScreeningMatches({ status, page, pageSize: PAGE_SIZE }));

  return (
    <div className="space-y-4">
      <PageHeader
        title="Screening"
        description="Possible matches against the UN, OFAC and your own lists. Confirm a match only when other details (ID number, date of birth, nationality) agree."
      />
      <Tabs
        label="Match status"
        items={VIEWS.map((v) => ({ href: `/screening?status=${v}`, label: STATUS_LABELS[v], active: v === status }))}
      />
      {query.error ? <ErrorState message={query.error} onRetry={query.reload} /> : null}
      {!query.data && !query.error ? <LoadingState /> : null}
      {query.data && query.data.items.length === 0 ? (
        <EmptyState>{status === "PENDING_REVIEW" ? "Nothing to review. New customers and list updates are screened automatically." : "No matches here yet."}</EmptyState>
      ) : null}
      {query.data && query.data.items.length > 0 ? (
        <DataTable<ScreeningMatchSummary>
          rows={query.data.items}
          rowKey={(m) => m.id}
          columns={[
            {
              header: "Customer",
              cell: (m) => (
                <Link href={`/screening/${m.id}`} className={linkClass}>
                  {m.customerName}
                </Link>
              ),
            },
            { header: "Listed as", cell: (m) => m.matchedName },
            { header: "List", cell: (m) => SOURCE_LABELS[m.source] },
            { header: "Category", cell: (m) => CATEGORY_LABELS[m.category] },
            { header: "Similarity", className: "tabular-nums", cell: (m) => `${scorePercent(m.score)}${m.birthYearMatched ? " · birth year agrees" : ""}` },
            { header: "Found", className: "whitespace-nowrap", cell: (m) => formatDateTime(m.createdAt) },
          ]}
          footer={
            <Pagination
              page={query.data.page}
              pageSize={query.data.pageSize}
              totalCount={query.data.totalCount}
              onPageChange={(p) => router.push(`/screening?status=${status}&page=${p}`)}
            />
          }
        />
      ) : null}
    </div>
  );
}
```

Add `useRouter` to the `next/navigation` import and `const router = useRouter();` at the top of `ScreeningQueue`.

- [ ] **Step 2: Match review page**

Create `src/app/(app)/screening/[id]/page.tsx`:

```tsx
"use client";

import Link from "next/link";
import { useParams } from "next/navigation";
import { FormEvent, useState } from "react";
import { Button, Field, inputClass } from "@/components/ui/Field";
import { PageHeader } from "@/components/ui/PageHeader";
import { ErrorState, LoadingState } from "@/components/ui/States";
import { confirmScreeningMatch, dismissScreeningMatch, getScreeningMatch } from "@/lib/api/screening";
import { Permissions } from "@/lib/auth/permissions";
import { useSession } from "@/lib/auth/useSession";
import { formatDateTime } from "@/lib/format/time";
import { errorMessage, useApiQuery } from "@/lib/hooks/useApiQuery";
import { CATEGORY_LABELS, scorePercent, SOURCE_LABELS, STATUS_LABELS, type ScreeningMatchDetail } from "@/lib/screening/match";

const linkClass = "text-[var(--accent-strong)] underline-offset-2 hover:underline";

export default function ScreeningMatchPage() {
  const { id } = useParams<{ id: string }>();
  const query = useApiQuery(`screening-match:${id}`, () => getScreeningMatch(id));
  if (query.error && !query.data) return <ErrorState message={query.error} onRetry={query.reload} />;
  if (!query.data) return <LoadingState label="Loading match…" />;
  return <MatchReview detail={query.data} onReviewed={query.reload} />;
}

function MatchReview({ detail, onReviewed }: { detail: ScreeningMatchDetail; onReviewed: () => void }) {
  const { can } = useSession();
  const { match, subject } = detail;
  const pending = match.status === "PENDING_REVIEW";

  return (
    <div className="grid gap-6 lg:grid-cols-[minmax(0,1fr)_20rem]">
      <div className="space-y-6">
        <PageHeader
          back={{ href: "/screening", label: "Screening" }}
          title={match.customerName}
          description={`${STATUS_LABELS[match.status]} · ${SOURCE_LABELS[match.source]} · ${CATEGORY_LABELS[match.category]} · found ${formatDateTime(match.createdAt)}`}
        />

        <section className="grid gap-4 sm:grid-cols-2">
          <div className="rounded-lg border border-[var(--line)] bg-[var(--panel)] p-4">
            <h2 className="text-xs font-medium uppercase tracking-wider text-[var(--muted)]">Customer</h2>
            <p className="mt-2 text-lg text-[var(--ink)]">
              <Link href={`/customers/${match.customerId}`} className={linkClass}>
                {match.customerName}
              </Link>
            </p>
            <p className="mt-1 text-sm text-[var(--muted)]">
              Name similarity {scorePercent(match.score)}
              {match.birthYearMatched ? " · birth year agrees with the list" : ""}
            </p>
          </div>
          <div className="rounded-lg border border-[var(--line)] bg-[var(--panel)] p-4">
            <h2 className="text-xs font-medium uppercase tracking-wider text-[var(--muted)]">Listed subject</h2>
            <p className="mt-2 text-lg text-[var(--ink)]">{match.matchedName}</p>
            <dl className="mt-2 space-y-1 text-sm">
              <Row label="Reference" value={`${SOURCE_LABELS[subject.source]} ${subject.externalId}`} />
              <Row label="Also known as" value={subject.names.filter((n) => n !== match.matchedName).join("; ") || "—"} />
              <Row label="Born" value={subject.birthYears.join(", ") || "—"} />
              <Row label="Nationality" value={subject.countries.join(", ") || "—"} />
              <Row label="Details" value={subject.detail ?? "—"} />
            </dl>
          </div>
        </section>

        {!pending ? (
          <section className="rounded-lg border border-[var(--line)] bg-[var(--panel)] p-4 text-sm">
            <p>
              <span className="font-medium text-[var(--ink)]">{STATUS_LABELS[match.status]}</span>
              {match.reviewedAt ? ` on ${formatDateTime(match.reviewedAt)}` : ""}: {detail.reviewReason}
            </p>
            {detail.alertId ? (
              <p className="mt-2">
                <Link href={`/alerts/${detail.alertId}`} className={linkClass}>
                  Open the alert raised by this match
                </Link>
              </p>
            ) : null}
          </section>
        ) : null}
      </div>

      <aside className="space-y-4 lg:sticky lg:top-20 lg:self-start">
        {pending && can(Permissions.ScreeningReview) ? <ReviewForm matchId={match.id} onReviewed={onReviewed} /> : null}
        {pending && !can(Permissions.ScreeningReview) ? (
          <p className="rounded-lg border border-[var(--line)] bg-[var(--panel)] p-4 text-sm text-[var(--muted)]">
            A reviewer decides on screening matches.
          </p>
        ) : null}
      </aside>
    </div>
  );
}

function Row({ label, value }: { label: string; value: string }) {
  return (
    <div className="grid grid-cols-[7rem_1fr] gap-2">
      <dt className="text-[var(--muted)]">{label}</dt>
      <dd className="text-[var(--ink)]">{value}</dd>
    </div>
  );
}

function ReviewForm({ matchId, onReviewed }: { matchId: string; onReviewed: () => void }) {
  const [reason, setReason] = useState("");
  const [pending, setPending] = useState<"confirm" | "dismiss" | null>(null);
  const [error, setError] = useState<string | null>(null);

  async function submit(event: FormEvent, decision: "confirm" | "dismiss") {
    event.preventDefault();
    if (!reason.trim()) {
      setError("Give a reason for the record.");
      return;
    }
    setPending(decision);
    setError(null);
    try {
      await (decision === "confirm" ? confirmScreeningMatch(matchId, reason) : dismissScreeningMatch(matchId, reason));
      onReviewed();
    } catch (err) {
      setError(errorMessage(err, "Review failed"));
    } finally {
      setPending(null);
    }
  }

  return (
    <form className="space-y-3 rounded-lg border border-[var(--line)] bg-[var(--panel)] p-4" onSubmit={(e) => submit(e, "dismiss")}>
      <h2 className="text-xs font-medium uppercase tracking-wider text-[var(--muted)]">Decision</h2>
      <Field label="Reason" hint="Kept in the audit log. Say which details agreed or differed.">
        <textarea rows={4} value={reason} onChange={(e) => setReason(e.target.value)} className={inputClass} />
      </Field>
      {error ? <p className="text-sm text-red-800">{error}</p> : null}
      <div className="flex flex-wrap gap-2">
        <Button variant="danger" disabled={pending !== null} onClick={(e) => submit(e, "confirm")}>
          {pending === "confirm" ? "Confirming…" : "Confirm match"}
        </Button>
        <Button type="submit" disabled={pending !== null}>
          {pending === "dismiss" ? "Saving…" : "Not the same person"}
        </Button>
      </div>
      <p className="text-xs text-[var(--muted)]">Confirming raises an alert and moves the customer's risk score.</p>
    </form>
  );
}
```

- [ ] **Step 3: Customer panel**

Create `src/components/screening/ScreeningPanel.tsx`:

```tsx
"use client";

import Link from "next/link";
import { useState } from "react";
import { Button } from "@/components/ui/Field";
import { EmptyState, ErrorState, LoadingState } from "@/components/ui/States";
import { listScreeningMatches, screenCustomer } from "@/lib/api/screening";
import { Permissions } from "@/lib/auth/permissions";
import { useSession } from "@/lib/auth/useSession";
import { errorMessage, useApiQuery } from "@/lib/hooks/useApiQuery";
import { CATEGORY_LABELS, scorePercent, SOURCE_LABELS, STATUS_LABELS } from "@/lib/screening/match";

const linkClass = "text-[var(--accent-strong)] underline-offset-2 hover:underline";

export function ScreeningPanel({ customerId }: { customerId: string }) {
  const { can } = useSession();
  const query = useApiQuery(`customer-screening:${customerId}`, () => listScreeningMatches({ customerId, pageSize: 20 }));
  const [message, setMessage] = useState<string | null>(null);
  const [pending, setPending] = useState(false);

  async function rescreen() {
    setPending(true);
    setMessage(null);
    try {
      const result = await screenCustomer(customerId);
      setMessage(result.newMatches === 0 ? "No new matches." : `${result.newMatches} new possible match(es).`);
      query.reload();
    } catch (err) {
      setMessage(errorMessage(err, "Screening failed"));
    } finally {
      setPending(false);
    }
  }

  return (
    <div className="space-y-3">
      {can(Permissions.CustomerWrite) ? (
        <div className="flex items-center gap-3">
          <Button onClick={rescreen} disabled={pending}>
            {pending ? "Screening…" : "Screen now"}
          </Button>
          {message ? <span className="text-sm text-[var(--muted)]">{message}</span> : null}
        </div>
      ) : null}
      {query.error ? <ErrorState message={query.error} onRetry={query.reload} /> : null}
      {!query.data && !query.error ? <LoadingState /> : null}
      {query.data && query.data.items.length === 0 ? <EmptyState>No matches on any screening list.</EmptyState> : null}
      {query.data && query.data.items.length > 0 ? (
        <ul className="divide-y divide-[var(--line)] rounded-md border border-[var(--line)] bg-[var(--panel)]">
          {query.data.items.map((m) => (
            <li key={m.id} className="flex flex-wrap items-center justify-between gap-2 px-3 py-2 text-sm">
              <Link href={`/screening/${m.id}`} className={linkClass}>
                {m.matchedName}
              </Link>
              <span className="text-[var(--muted)]">
                {SOURCE_LABELS[m.source]} · {CATEGORY_LABELS[m.category]} · {scorePercent(m.score)} · {STATUS_LABELS[m.status]}
              </span>
            </li>
          ))}
        </ul>
      ) : null}
    </div>
  );
}
```

In `customers/[id]/page.tsx`, after the Risk section:

```tsx
{can(Permissions.ScreeningRead) ? (
  <Section title="Screening">
    <ScreeningPanel customerId={customer.id} />
  </Section>
) : null}
```

(import `ScreeningPanel`; take `can` from `useSession()` if the page doesn't already). Also show `customer.dateOfBirth` in the header description when present.

- [ ] **Step 4: Alert page link**

In `alerts/[id]/page.tsx`, inside the "Why this alert fired" section after the dismissal paragraph:

```tsx
{isScreeningAlert(alert.additionalContext) && typeof alert.additionalContext.screeningMatchId === "string" ? (
  <p className="mt-3 text-sm">
    <Link href={`/screening/${alert.additionalContext.screeningMatchId}`} className="text-[var(--accent-strong)] underline-offset-2 hover:underline">
      Open the screening match
    </Link>
    {typeof alert.additionalContext.reviewReason === "string" && alert.additionalContext.reviewReason
      ? ` · confirmed because: ${alert.additionalContext.reviewReason}`
      : ""}
  </p>
) : null}
```

and hide the "Related transactions" section for screening alerts (they have none): extend its condition with `&& !isScreeningAlert(alert.additionalContext)`.

- [ ] **Step 5: List admin page**

Create `src/app/(app)/admin/screening/page.tsx`:

```tsx
"use client";

import { FormEvent, useState } from "react";
import { DataTable } from "@/components/ui/DataTable";
import { Button, Field, inputClass } from "@/components/ui/Field";
import { PageHeader } from "@/components/ui/PageHeader";
import { ErrorState, LoadingState } from "@/components/ui/States";
import { getScreeningLists, refreshScreeningLists, uploadTenantList } from "@/lib/api/screening";
import { formatDateTime } from "@/lib/format/time";
import { errorMessage, useApiQuery } from "@/lib/hooks/useApiQuery";
import { SOURCE_LABELS, TENANT_LIST_TEMPLATE, type ScreeningLists } from "@/lib/screening/match";
import { validateImportFile } from "@/lib/transactions/importFile";

type SourceRow = ScreeningLists["sources"][number];

export default function ScreeningListsPage() {
  const query = useApiQuery("screening-lists", getScreeningLists);
  const [notice, setNotice] = useState<string | null>(null);
  const [error, setError] = useState<string | null>(null);
  const [busy, setBusy] = useState<"refresh" | "upload" | null>(null);
  const [file, setFile] = useState<File | null>(null);

  async function refresh() {
    setBusy("refresh");
    setError(null);
    setNotice(null);
    try {
      const result = await refreshScreeningLists();
      const summary = result.results.map((r) => `${r.source}: ${r.status.toLowerCase()}${r.error ? ` (${r.error})` : ""}`).join(" · ");
      setNotice(`${summary}. ${result.newMatches} new possible match(es).`);
      query.reload();
    } catch (err) {
      setError(errorMessage(err, "Refresh failed"));
    } finally {
      setBusy(null);
    }
  }

  async function upload(event: FormEvent) {
    event.preventDefault();
    if (!file) return;
    const invalid = validateImportFile(file);
    if (invalid) {
      setError(invalid);
      return;
    }
    setBusy("upload");
    setError(null);
    setNotice(null);
    try {
      const result = await uploadTenantList(file);
      setNotice(`List replaced with ${result.entryCount} entries. ${result.newMatches} new possible match(es).`);
      query.reload();
    } catch (err) {
      setError(errorMessage(err, "Upload failed"));
    } finally {
      setBusy(null);
    }
  }

  const templateHref = `data:text/csv;charset=utf-8,${encodeURIComponent(TENANT_LIST_TEMPLATE)}`;

  return (
    <div className="space-y-6">
      <PageHeader
        title="Screening lists"
        description="The UN and OFAC lists refresh every night; every customer is rescreened when a list changes."
        actions={
          <Button onClick={refresh} disabled={busy !== null}>
            {busy === "refresh" ? "Refreshing…" : "Refresh now"}
          </Button>
        }
      />
      {notice ? <p className="rounded-md bg-emerald-50 px-3 py-2 text-sm text-emerald-900">{notice}</p> : null}
      {error ? <ErrorState message={error} /> : null}
      {!query.data && !query.error ? <LoadingState /> : null}
      {query.error ? <ErrorState message={query.error} onRetry={query.reload} /> : null}

      {query.data ? (
        <DataTable<SourceRow>
          rows={query.data.sources}
          rowKey={(s) => s.source}
          columns={[
            { header: "List", cell: (s) => SOURCE_LABELS[s.source] },
            { header: "Entries", className: "tabular-nums", cell: (s) => s.entryCount.toLocaleString() },
            { header: "Published", cell: (s) => (s.publishedAt ? formatDateTime(s.publishedAt) : "—") },
            { header: "Loaded", cell: (s) => formatDateTime(s.loadedAt) },
          ]}
        />
      ) : null}

      <section className="space-y-3 rounded-md border border-[var(--line)] bg-[var(--panel)] p-4">
        <h2 className="text-lg font-semibold text-[var(--ink)]">Your list</h2>
        <p className="text-sm text-[var(--muted)]">
          {query.data?.tenantList.entryCount
            ? `${query.data.tenantList.entryCount} entries, uploaded ${formatDateTime(query.data.tenantList.uploadedAt!)}.`
            : "No list uploaded yet."}{" "}
          Use it for politically exposed persons, the Kenya domestic sanctions list and your internal blacklist. Uploading replaces the whole list.
        </p>
        <form onSubmit={upload} className="flex flex-wrap items-end gap-3">
          <Field label="CSV file" hint="Columns: reference, name, category (PEP, SANCTIONS, WATCHLIST), kind (INDIVIDUAL, ENTITY); optional aliases, birthYear, country, notes.">
            <input type="file" accept=".csv,text/csv" onChange={(e) => setFile(e.target.files?.[0] ?? null)} className={inputClass} />
          </Field>
          <Button type="submit" variant="primary" disabled={!file || busy !== null}>
            {busy === "upload" ? "Uploading…" : "Replace list"}
          </Button>
          <a href={templateHref} download="aegis-screening-list-template.csv" className="text-sm text-[var(--accent-strong)] underline-offset-2 hover:underline">
            Download template
          </a>
        </form>
      </section>
    </div>
  );
}
```

- [ ] **Step 6: Run checks and build**

Run: `npm test && npm run lint && npx tsc --noEmit && npm run build`
Expected: all pass.

- [ ] **Step 7: Check in the browser (macOS)**

With the API on 5092 (screening refresh enabled, internet available) and `NEXT_PUBLIC_AEGIS_API_BASE_URL=http://localhost:5092 npx next start -p 3000`:
1. Sign in as `admin@smoke-6121.test`. Open **Screening lists**: UN about 1,000 and OFAC about 17,600 entries.
2. Upload the template CSV, then create a customer named "Jane Wanjiku Doe" through the API (`POST /api/v1/customers`). **Screening** shows a pending PEP match with a badge count in the sidebar.
3. Open the match, confirm with a reason: the alert link opens a HIGH "Confirmed PEP match" alert whose headline names the list; the customer's risk panel shows PEP points.
4. Sign in as `daniel@smoke-6121.test` (Viewer): Screening is visible, with no decision form.
5. Check at 1024 px and 800 px for overflow.

- [ ] **Step 8: Commit**

```bash
git add src
git commit -m "feat(screening): review queue, match review, customer panel and list admin pages"
```

---

### Task 11: Demo data, docs, review

**Files:**
- Modify: `aegis/scripts/seed-volume.py` (screening step), `aegis/README.md` (seeder note)
- Modify: `aegis-docs/plans/2026-10-01-aegis-build-roadmap.md` (Phase 3 status), `aegis-docs/README.md` (document list)

- [ ] **Step 1: Seeder screening step**

Add a `screening(api, token, tag)` step to `seed-volume.py`, run after customers are created and before analyst work:
1. `PUT /api/v1/screening/tenant-list` with a CSV of six fictional entries: three PEPs and two watchlist entries matching seeded customer names, and one PEP matching nobody.
2. Create two individuals whose names match UN-listed people, taken from the loaded list by refreshing first (`POST /screening/lists/refresh`), so the demo shows real list hits. Use names already public on the UN list; dates of birth inside the listed years.
3. Review some matches: confirm one PEP and one UN match (reason "Seeded: ID number matches"), dismiss one watchlist match ("Seeded: different date of birth"), leave the rest pending.

Keep it stdlib-only like the rest of the script, use multipart form encoding for the upload, and gate it behind `--screening` (default on).

- [ ] **Step 2: Docs**

- Roadmap Phase 3: mark 3.1–3.4 done with one line each pointing at this plan, and record the decisions at the top of this file.
- `aegis-docs/README.md`: add this plan to the document list.
- `aegis/README.md`: list the `Screening:*` settings (`Refresh:Enabled`, `Refresh:HourUtc`, `Sources:{UN|OFAC}:Url|Path`, `MatchThreshold`) and `Risk:SyncScreeningFactorsOnStartup`.

- [ ] **Step 3: Full verification**

```bash
cd aegis && dotnet test tests/Unit && dotnet test tests/Integration && dotnet test tests/EndToEnd && dotnet test tests/Architecture
cd ../aegis-console && npm test && npm run lint && npx tsc --noEmit && npm run build
```

Expected: everything passes.

- [ ] **Step 4: Commit and request review**

```bash
git add -A && git commit -m "chore(screening): demo data and docs"
```

Start a separate review session (new chat) on the `phase-3-screening` branches of `aegis` and `aegis-console` before merging.

---

## Self-review

- **PRD §31 coverage:** customer → screening request (`ScreenCustomerAsync`/`ScreenTenantAsync`, Task 6) → provider (UN, OFAC, tenant list; `IWatchlistDownloader` and the parsers keep vendors out of the domain, Tasks 2, 3, 5) → match results (`ScreeningMatch`, Task 4) → match review (Tasks 7, 10) → decision (confirm or false positive, audited, Task 6). Screening hits feed the risk engine (Task 8). Roadmap items 3.1–3.4 map to Tasks 1–6, 2/5, 6–8 and 9–10.
- **Audit:** match created, match resolved, alert created, tenant list uploaded. Global list loads are logged, not audited, because audit events are tenant-scoped.
- **Isolation:** tenant filters on `TenantWatchlistEntry` and `ScreeningMatch`; every repository query also filters by `TenantId`; the isolation test is in Task 7.
- **Type names used across tasks:** `WatchlistSubject`, `ScreeningCandidate`, `PotentialMatch`, `WatchlistIndex.DefaultThreshold`, `ScreeningMatch.Confirm(reviewer, reason, alertId)`, `ScreeningTriggers.*`, `ScreeningAlertSource.RuleId/RuleCode`, `RiskTriggers.ScreeningConfirmed`, `RiskActor` (reused from `Aegis.Application.Risk`).
- **Known follow-ups (not in this phase):** customer update endpoint (date of birth and name changes), EU/UK lists, country-name-to-ISO matching, delta rescreening for large customer bases, maker-checker on confirmations.
