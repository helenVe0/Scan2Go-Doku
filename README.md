# Scan2Go

Scan2Go answers one question: **when do I have to leave home to catch my train?**

You can find the demo video here: https://www.loom.com/share/98d0a65abddb4cdb90440240a35cfaaf

The idea is a screen in a public place. It shows a QR code, you scan it with your phone and
type in your home address, then pick your departure station, your destination, the day and
the time you want to arrive. A few seconds later the screen shows one big number: when you
have to walk out of your door. Everything the user does happens by scanning QR codes, so
the display itself needs neither a keyboard nor a touchscreen.

## Where everything lives

| Part | Where |
|---|---|
| Process model | `FINALE_Abgabe.xml` (CPEE testset) |
| Display pages | `~/public_html/` → `https://lehre.bpm.in.tum.de/~go35vom/` |
| Own service | `~/scan2go_endpoints/main.py`, port 12349 |
| Station data | `~/scan2go_endpoints/db-stations.ndjson`, 5388 German stations |

| Endpoint | URL |
|---|---|
| `frames_init` / `frames_display` | `https://cpee.org/out/frames/{helen}/` |
| `search_stations` | `https://lehre.bpm.in.tum.de/ports/12349/search-stations` |
| `geocode` | `https://nominatim.openstreetmap.org/search` |
| `get_connections` | `https://api.transitous.org/api/v1/plan` |
| `get_local_route` | `https://lehre.bpm.in.tum.de/ports/12349/local-route` |
| `calculate_departure` | `https://lehre.bpm.in.tum.de/ports/12349/calculate-departure` |
| `timeout` | `https://cpee.org/services/timeout.php` |

## One session, step by step

The process starts with `Init Frame`, which reserves a 1×1 frame on the display, and then
runs exactly one session and ends.


| Step | Call | Writes |
|---|---|---|
| Display welcome page | `welcome.html` | `home_address` |
| Get train stations nearby | `search-stations?address=<home_address>` | `stations` |
| Display selection of train stations | `stations.html` | `station_selected` |
| Get geocode from home station | Nominatim | `coordinates_homestation` |
| Display Cities | `cities.html` | `destination` |
| Display Dates | `days.html` | `date` |
| Display Times | `times.html` | `time`, `arrival_time` |
| Geocoding | Nominatim | `coordinates_destination` |
| Get train connections | Transitous `/plan` | `raw_connections` |
| Get best connection | script task | `best_connection` |
| Get local route | `local-route` | `travel_time_to_station` |
| Calculate departure | `calculate-departure` | `departure_from_home` |

**The welcome page** shows a QR code that points at `address.html` with the CPEE callback
attached. The address is the only thing the user has to type, so it is the only page that
runs on the phone; everything after that is scanning.

**The station lookup** takes the whole address and returns up to six stations, and the user
picks one by scanning. 

**Both geocodings** go to Nominatim, one for the station the user picked, one for the
destination. They store `nil` when Nominatim has no match, which is what the check behind
each call looks for. The shorter `result[0]["lat"]` cannot do that: on an empty answer it
raises before anything is stored, so the check has nothing to look at.

**Date and time** are two separate pages because eight times times seven days would be 56
QR codes on one screen. The time page also builds `arrival_time`, the timestamp the
connection search needs:

```ruby
data.arrival_time = Time.local(y, m, d, h, min).strftime("%Y-%m-%dT%H:%M:%S%:z")
```

The offset matters. Transitous refuses a timestamp without one, and if you just append `Z`
it reads 21:00 as 21:00 UTC, which in summer looks for a train two hours too late.
`Time.local` takes the offset from the engine host, so summer and winter time are handled
without me writing a rule for it.

**The connection search** asks Transitous with `arriveBy=true`, five itineraries and
`transitModes=LONG_DISTANCE,HIGHSPEED_RAIL,RAIL,BUS`. `BUS` is in the list so a bus feeder
to the station is allowed, the way a normal timetable app would route it. The response is
large, so the finalize keeps only six fields per itinerary. Two of them are computed:

```ruby
transit = c["legs"].reject { |l| l["mode"] == "WALK" }
"train_departure" => transit.first["startTime"],
"train_arrival" => transit.last["endTime"],
```

An itinerary does not begin at the station, it begins at the coordinate Nominatim returned,
and its first leg is the walk to the platform. So `startTime` is a few minutes before the
train actually leaves, and everything downstream would be that much too early. The first
and the last leg that is not a `WALK` are the train the user has to catch.

**Picking one connection** is a script task, because it is a few lines of Ruby and does not
need the network:

```ruby
connection_candidates = data.raw_connections.select { |c| (c["transfers"] || 99) <= data.max_transfers }
connection_candidates = data.raw_connections if connection_candidates.empty?

latest = connection_candidates.map { |c| c["train_departure"] }.max
connection_candidates = connection_candidates.select { |c| c["train_departure"] == latest }

best = connection_candidates.min_by { |c| c["duration"] }
```

At most three changes, and if that leaves nothing at all the filter is ignored rather than
failing. Then the latest departure wins, because the big number on the screen is the time
the user has to leave, and every candidate arrives in time anyway. Then, among the
connections that leave at that same minute, the fastest one. That last step is there
because sorting only by transfers buys long layovers: in one test the result had a full
hour of waiting in Munich, while the alternative with one change more was faster.

**The last two calls** work out the way from the front door to the station and turn
everything into one time. Both get `buffer_minutes` from the process data, 15 by default,
so the time reserved for finding the platform exists in one place and can be changed in the
CPEE editor.

## The watchdog, and what happens when something fails

The planning above is one branch of a parallel block. The other branch is a small loop that
waits three seconds and then runs one line:

```ruby
data.timeout_occurred = true if (Time.now.to_i - data.last_interaction) > 60
```

Every page the user interacts with refreshes `last_interaction`, so the loop keeps ticking
while somebody is using the display and stops when nobody has scanned anything for a
minute. The parallel block runs with `wait="1"` and `cancel="last"`, which means it
continues as soon as one branch is done and cancels the other. Either the planning finishes
and the result appears, or the watchdog gives up on the user and the error page appears.
Without that, the display would sit on a page nobody is looking at until someone restarts
it.

Five calls can come back without something usable: the station lookup, both geocodings, the
connection search and the local route. Each has a gateway with a default flow, and every
error path does the same two things:

```ruby
data.error_occurred = true
data.last_interaction = ( Time.now.to_i - 60 )
```

Backdating the timestamp makes the watchdog trip on its next tick, so it finishes and
cancels the planning branch. The ten second wait behind each error
path is only there to outlast that, otherwise the planning would carry on with an empty
result and, for example, geocode the station from the previous session.

After the join one gateway decides what to show:

```
data.error_occurred != true && data.timeout_occurred != true   →  result.html
otherwise                                                      →  error.html
```

Both flags are reset at the start of every session, right after the welcome page.

## The Python service

Three things were easier outside the process: searching a 5388 station data file, talking
to Google Routes with a field mask, and doing date arithmetic. They live in one Bottle file
on `::0:12349`, which the course server exposes as
`https://lehre.bpm.in.tum.de/ports/12349/`.

* **`/search-stations`** takes the typed address, keeps the part after the last comma
  without the postal code, and returns up to six stations from that city. `Hbf`,
  `Hauptbahnhof` and `Ost/West/Nord/Südbahnhof` count as main stations. Small towns have
  none of those, so the fallback is the station with the highest `weight` in the dataset,
  which for a town with a single stop is exactly the right answer. Sorted busiest first,
  and six is what fits on the screen as QR codes.
* **`/local-route`** asks Google Routes twice, once walking and once by public transport,
  for the way from the front door to the station, aiming to be there `buffer_minutes`
  before the train. Only the transit call gets an arrival time, because walking does not
  depend on a timetable. Each mode has its own try/except, and `recommended_minutes` is the
  smaller of the two that answered.
* **`/calculate-departure`** is the arithmetic: train departure minus the buffer minus the
  travel time, returned as UTC and as local `HH:MM`.

The station list is read once at startup instead of per request, since the file does not
change while the service runs. Every response is JSON with `ensure_ascii=False` so that
`München` stays readable on the display. And errors come back as `{"error": …}` with status
200 rather than a 4xx: the process should decide what an empty result means, not the HTTP
layer, and a failed call in CPEE is harder to reason about than a body with a missing
field.

The Google key comes from `~/scan2go_endpoints/env` through python-dotenv, so it stays out
of `public_html` and out of this repository.

## The display pages

The frames service loads each page into the display frame and posts the process data into
it. The pages answer by showing a QR code that points at
`send.php?info=<value>&cb=<callback>`. The phone opens that URL, the relay forwards the
value to the callback, and the waiting CPEE call continues with it. That is why the phone
never talks to CPEE directly.

| Page | What it shows |
|---|---|
| `welcome.html` | headline, four steps, QR code to `address.html` |
| `address.html` | the only form, opened on the phone: street, postal code, city |
| `stations.html` | one QR per station from `data.stations` |
| `cities.html` | one QR per entry in `data.cities` |
| `days.html` | the next seven days, generated in the page |
| `times.html` | eight arrival times, the ones already gone greyed out |
| `result.html` | departure time, travel time, transfers, duration, the connection |
| `error.html` | "Nothing Found", says it returns to the start |

`welcome.html` and `days.html` use CPEE's `frames.js`. The four pages that need process
data at render time carry a six line `postMessage` listener of their own instead, with an
origin check on `tum.de` and `cpee.org`. That way they depend on neither jQuery nor an
external script, which is one thing less that can fail on a screen nobody is standing next
to.

`times.html` gets `selected_date` so it can grey out the times that have already passed
today. It sends back only the time, never the date, so the value the process receives is
the same whether or not `selected_date` arrives.

The destination list lives in the process data as `cities`, not in `cities.html`, so it can
be changed in the CPEE editor without touching the web space. One entry needed care: plain
`Bern` geocodes to the city boundary rather than the station, so the id is `Bern Hbf` while
the name on the screen stays `Bern`.

`result.html` is the only page that formats anything. It turns the UTC timestamps into
local time and seconds into `6h 31min`, and falls back to `--:--` and `--` for missing
fields, so a half filled result still renders instead of showing a broken screen.
