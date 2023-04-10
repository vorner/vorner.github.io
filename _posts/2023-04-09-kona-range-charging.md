# The Kona EV consumption, range and charging

As noted in the [previous post], I'm using a Kona EV for almost a year now, so I
can write about the experience with it.

It's a 2022 model with the smaller battery (39.2 kWh), 11kW charger and heat
pump. There's also a 64 kWh battery option which, according to reports of other
people, is much better for long trips.

However, I think even this smaller battery shouldn't be just dismissed. Sure,
there are compromises about it, but it's far from a car that „can't get
anywhere“.

## Consumption

Consumption is a strong point of this car. Hyundai boasts quite low numbers and
I can confirm that they are, in a sense, realistic.

That is, if you drive in summer conditions on dry roads, you can get these
numbers on a mixed road trips, or even go lower, without any particular
changes to driving style (that is, driving with the flow of the rest of the
traffic, not 50km/h on a highway and not drafting behind trucks).

I've had few trips that included some amount of highway ‒ about 20% (highways
are the worst category for EVs as consumption goes) that went as low as
10kWh/100km (this is significantly lower than the official numbers).

On the other hand, EV consumption ‒ this car included ‒ is sensitive to many
factors and vary a lot. During wet winter day on a mostly-highway drive, I was
able to get over 20kWh/100km.

While both are somewhat extremes, the difference is twofold.

### Range estimates

The car estimates how much range is left in the battery. This is based on recent
_past_ driving. It doesn't take the planned trip into account in any way (in
this trim, it doesn't even have built-in navigation, so it has no chance of
knowing where it is going).

This means, if driving style changes, the estimate is wrong and the driver
should know not to rely on it for planning directly.

Most daily drives are within a safe range with enough margin and each owner
learns fast not to worry about them.

The range estimate is useful for keeping an eye while driving long trips and
trying to get as much from a single charge as possible. Simply comparing the
estimate with the distance. While the estimate might be wrong initially, it'll
get better and the margin will either increase or decrease. This gives an early
warning that it might not be enough to get to the destination and a charging
stop earlier might be needed.

For estimating and planning longer trips, one can use a tool like [ABRP]. But it
has somewhat conservative information about the consumption of this car (maybe
even about other cars). I've adjusted the „Reference consumption“ to 150 on
summer tires and to 175 on winter tires. I suspect the car is sold in larger
numbers somewhere in Scandinavia with different tires and this offsets the
tool's estimates.

### Non-linear percents

One quirk worth knowing is, the battery percents are not based on kWh left in
the battery, but on Ah in it. This may look like an overly technical detail, but
the result of that is that a percent of battery at the top (eg. between 100% and
99%) means more distance than the percent at the bottom. This is because the
voltage of the battery drops with the state of charge.

In other words, the percents are disappearing faster when the battery is near
empty.

The estimator showing kilometres tries to account for that.

### Factors influencing the consumption

There are many factors influencing the consumption. However, some of them seem
to be different from common „EV folklore“, at least according to my
observations.

#### Speed

Driving speed is likely the biggest factor. The effect is almost not observable
to about 70km/h and is small to maybe 100km/h, then it rises more prominently.

There seems to be a significant rise somewhere between 120 and 130. There's also
an increase of noise at about these speeds.

Considering the highway speed in Korea is 120km/h, I suspect they tuned the
aerodynamics for „their“ speeds and somehow neglected our European top speeds
(130 is actually on the high end of allowed speeds across the world).

Lowering speed on the highway can save some energy, lowering speed in city
driving likely not.

#### Tires

As noted indirectly above, winter vs summer tires have a significant influence
on the consumption.

#### External addons

Like, roof boxes, bicycles on the roof, etc. These all break the aerodynamics
and make the consumption worse.

I've heard that it's often better to put skis inside a roof box than carry them
bare on the roof rack ‒ even if bigger, the aerodynamics of the roof box is
slightly better.

#### Weather effects

Wind has a lot of effect ‒ that's obvious for a headwind, but even crosswind
makes the consumption higher.

Humidity also plays a role. Wet roads increase consumption, so does fog or rain
(that increases the aerodynamic drag) or simply humid air. This has more effect
in higher speeds.

What _doesn't_ seem to have that much of an effect is temperature. There seems
to be some small amount of consumption increase with cold, but all that humidity
and wind seems to be more significant factor. Nevertheless, during cold days,
one oftentimes get more humidity and more wind. Dry, sunny but cold day is going
to be more favorable than a summer rainstorm.

There's some energy lost for heating the cabin. With the heat pump, it's not
huge amount (the initial heating takes quite a bit, but keeping it warm during a
longer drive no longer consumes much) and it can be lowered even further by
using the Driver only option (which blows the warm air only through some of the
vents close to the driver seat, not wasting it on the rest of the empty car).
Lowering the temperature and using a heated seat can be used too, but it doesn't
seem to be a significant difference ‒ I use such setup more to keep myself alert
and awake than to save energy. All in all, the difference is not significant
enough to compromise comfort.

#### Air flaps

The car has air flaps at the front. Most of the time they are closed off, making
the car more aerodynamic. They get opened only on occasion when it needs to feed
a lot of air through the heat exchanger, either because of need to cool down or
to warm things using the heat pump.

They can't be operated directly by the driver and the only indication of if they
are open is higher consumption at highway speeds and a kind of very silent
„turbine noise“ coming from that area.

I'm going to leave some extensive observations about these and battery thermal
management for a separate post, as all that was getting somewhat long.

### Weight

Despite all the advice to keep clutter at home because of consumption, it seems
extra weight does _not_ influence the consumption in any measurable way (as long
as that weight is closed _inside_ the car, of course).

Considering how aggressive Hyundai cars are about using regenerative breaking,
it seems they are able to regain most of what extra is used when going uphill
and accelerating by going downhill and decelerating.

In other words, don't expect your range to suffer by giving another person a
ride, not even speaking about the effect of leaving the charging cable at home.

## AC charging

The car is equipped with a 11kW charger (3 phase x 16A). If connected to a
single-phase AC station, it can take up to 1x32A, which is about 7kW.

For those who don't know the difference, with AC charging the car is connected
to Alternating Current from the mains directly and the charger is inside the car
under the hood somewhere. This is the same for public slow charging stations and
by connecting the car to a power outlet (either the ordinary single-phase one,
which charges slower, or the red three-phase one many people have in their
garages).

Combined with the low consumption, the car can effectively „surf AC chargers“.
If I have an errand day and expect it would be more total driving than the full
range (which happened only once so far), I can plug the car to an AC charger at
every stop that has one available. I won't spend any extra time just for
charging and it would provide the extra needed to make the whole day run. Each
30 minute charging session counts.

Similar strategy can be used if such a car lives „on the street“ without its own
garage. When I was still living in a flat, most of the time the car was kept
charged by surfing charges, only occasionally I'd have to put it further away
from the house to a charger for a longer time (usually before or after a longer
trip).

## DC charging

The car is capable of DC fast charging.

Again, for people without the EV experience, in this case the charger lives in
the station. The charger inside the car is bypassed and the battery is fed
directly from the station by Direct Current. The station can contain a bigger
and heavier charger than would be practical for the car to carry around, which
allows the charger to be significantly more powerful. Using them is usually more
expensive (while an AC station is mostly just the plug, some fuses and a meter,
this is actually an expensive equipment). Most EV drivers use these only when in
hurry and most charging happens on the AC while parked for longer periods (like
overnight).

The car is willing to take up to about 46kW and a 10-80% charge is supposed to
take 48 minutes. These are not top of the game numbers now, but it's definitely
faster than the up to 11kW of AC charging.

The 64kWh battery version of this car takes the same time, it has higher max
power input. This is quite common phenomenon with cars that have smaller and
larger battery option. It's because the larger battery is made of the same
cells, just having more of them and all the cells are charged in parallel ‒
therefore, more cells can take more total power.

For comparison, Hyundai Ioniq 5/6 boast charge time of 18 minutes and something
over 200kW of max power input. As a reference, a water kettle is usually about
2kW.

### Charging curve

As with any EV on the market, the car is not willing to always take the max
power. The charge slows down with increasing state of charge ‒ and that's why
nobody really advertises how long the car takes to charge to 100%. Usually, it
takes about the same time to charge it from 80 to 100% as it takes from 0 to
80%.

But it's not like there would be a single steep drop at 80%. This car has a drop
at about 65% to some 35kW. Another drop is somewhere around 75% (I'm not sure
exactly where). Past some 87%, it charges at 13kW and will drop to about 6kW at
95% or so (it'll also drop to lower power on AC at that point).

This is for the smaller battery _with a heat pump_. While the large battery
always has the heat pump, there are versions of the smaller-battery one without
the pump. As I hear the pump running during a DC charge, I assume it is used for
more effective cooling. Without the heat pump, the only option is to circulate
the coolant through the radiator and blow air through it ‒ which might (or might
not) offer worse performance.

### Poor man's advantages of slow-ish DC charging

As mentioned, these are not really top of the game numbers any more. These are
not terrible, considering the consumption ‒ it would be worse if the car
consumed twice as much (it would also make the range on the same battery half
what it is).

Anyway, these conservative charging speeds have two small advantages.

Most of the chargers in Czech republic are 50kW ones. Due to the battery voltage
and limit of 125A on the cable (the common setup of the 50kW chargers), these
limit the car to some 44kW, which is very close to its limit. This means I can
just pick any charger based on amenities available around for me than adjusting
to the needs of the car.

With more conservative charging speed, the car is less sensitive to battery
temperature. The car doesn't have an option to preheat the battery before
reaching the charger, but it mostly doesn't need it. If the car is connected to
a DC charge first thing on a cold morning, it'll limit the charging speed for
few minutes (before it heats up by charging), but an hour of driving is enough
to warm it even during winter. This means, if one starts a trip with reasonably
charged car, the battery would be warm by the time charging is needed.

### Occasional failed handshakes

From time to time, the handshake between the car and the DC charger fails. It's
often the case with ABB chargers (I've never seen it on an AC charge, but the
communication protocol for these are significantly simpler and less fragile).

I've observed that this is likely related to the car having the charge port in
somewhat unusual location (it's on the nose and quite close to the ground).
Often, one needs to twist the cable, because the operator expected the port to
be from the side of the car, not from the front.

I've always managed to make the handshake work by either holding the connector
in place during the handshake or twisting the cable the other way around. Don't
give up on the first attempt and drive away.

### Long-distance travel

There are several ranges that come into play around an EV.

Even during winter, I assume a place 100km (a bit more in summer) away is within
a round-trip range ‒ if I charge to 100% before leaving, I'm almost sure to get
back home without a need to charge. In winter, this _might_ mean I'd have to
drive slightly slower on a highway (if the weather is bad and most of the drive
is on the highway) and that I might arrive home near empty (but that's OK as
long as I'm sure to get home, I have a charger there).

Similarly, I can reach a place about 200km (more in summer) away if there's a
destination charger there and stay there for a while, so I can go back again.
The presence and reliability of destination chargers is not always sure,
therefore one might want to have a backup plan.

Then there's something I'd call a comfortable range. It's around maybe 300km
away for me. This already requires a stop to charge, but I'm likely to want a
stop myself. I don't need a full recharge (or to 80% or so), only maybe 100km,
the charging would take about 20-25 minutes. That might be a bit longer than
necessary for a minimal toilet break, but about OK to also buy a cup of tea
(it's surprising how long a „just stop for a toilet“ actually takes in any car).

As the car has active cooling of the battery, it can do arbitrary number of
DC charging sessions in a row. That makes it possible driving long distances, at
least in theory. After the first longer leg (as it can start with 100% charge),
the optimum lies in something like 120-180km of highway driving and 30-40
minutes of charging. I also suspect it might be better to lower the driving
speed to something like 120km/h ‒ lower consumption means less of charging and
could mean arriving at the final destination sooner.

On a trip from Czech republic to Croatia (common summer holiday destination and
kind of a benchmark/argument of many people why EVs „suck“), there might be some
5 or 6 stops. That's probably somewhat on the discomfortable side of things. I
mean, people were willing to make these journeys with several hours of border
control queues, but one would still desire for something better.

I'd also note that there are EVs that do better ‒ including the bigger battery
variant of Kona, which (according to several tests) can get there with just
two stops ([1], [2]).

This is likely the lowest-end EV that's even _capable_ of this kind of
long-distance travel. Cheaper EVs either don't have active battery cooling ‒
and won't be able to do more than one or two DC fast charges in a single day
(further charge would be incredibly slow due to protection of the already hot
battery). Or, some even don't have the option to fast-charge.

### En-route adjustments

There's one trick worth knowing. If you know you'll not reach the destination,
but only by a small bit, it's either possible to stop and charge (in most
places, there's at least one charger every 30km or so ‒ this is with exceptions,
though, and it is worth having a look at them before making a longer trip). The
alternative is slowing down a bit, especially on the highway. Such thing will
lower the consumption and sometimes make it possible to get to the previously
planned destination faster than by stopping to charge.

Watching the estimated range and adjusting speed based on that is something that
can be done and this is in part how some EV drivers surprisingly often arrive
home with below 5% of battery, but never get stranded with 0% on the way.

## Charging infrastructure

### DC infrastructure

My personal opinion on how good or bad the infrastructure is here in Czech
republic. I'd say there's mostly enough coverage of DC chargers around. My view
is likely influenced by the fact my car is well satisfied with all these 50kW
chargers, though. And of course, as more EVs appear on the roads, we still will
need more.

However, not every charger out there has reasonable amenities around. As the
expected stay of a driver is 30 minutes or longer on a 50kW charger, a charger
that doesn't even have a toilet there is kind of useless.

Against what general non-EV population thinks, chargers aren't commonly broken
(sometimes glitchy ‒ see the note about failed handshakes) and it's easy to pick
a location with more than one stall, where the chance of all chargers being in
use is rather low. That is, reaching an occupied or broken charger isn't a
common experience.

### AC infrastructure

This is where I believe we need more infrastructure. A car is parked somewhere
most of its life and usually, when I drive somewhere, I spend at least a little
time there. This time can be spent charging ‒ but only as long as there's a
plug, AC station or something like that.

Shopping malls are mostly covered already by these. But we still lack them at
other places where people spend time ‒ ZOOs, castles, hotels, restaurants, ski
lifts. And near estates, block houses and other places where people tend to live
‒ both for the people there and for visitors. This is, to some extend, covered
in Prague, but other cities still have a lot to catch up.

Optimally, there should be a charger on every street and bigger parking lot.
Having a destination charger available in a destination basically doubles the
range at which there's no need to charge on the route and makes cheaper
small-battery cars more useful.

All in all, while we need some high-power DC charges by the highways and main
routes, it's possible to build significantly more AC chargers for the same money
and would be a better investment. It's just, opening a 300kW charger is
something that can be shown in news and gains publicity, while installing 20
power outlets on 10 different parking lots is not as glamorous to boast about.

[previous post]: {% post_url 2023-04-08-kona-ev-tips %}
[ABRP]: https://abetterrouteplanner.com/
[1]: https://www.youtube.com/watch?v=HEOVSIA0eZY
[2]: https://www.youtube.com/watch?v=ZAsNUiUHGC8
