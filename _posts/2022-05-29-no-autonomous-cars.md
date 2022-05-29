# Why we won't have fully autonomous cars soon

As I've been browsing the Internet when researching EVs and cars in general
(because I've wanted to [buy one]), I've came across a lot of speculations and
hopes about future autonomous cars. There's a lot of buzz coming from Tesla fans
specifically, even though other car makers have some ambitions too.

I haven't tracked each claim down to its origin, but these are something I've
heard from various places, so let's say these are the Internet „folklore“. Here
are two picks:

* Tesla might start selling cars without a steering wheel soon, probably in
  China first.
* Soon, possibly this year, Teslas with full self-driving package will be able
  to act as robo-taxis. That way owners may release the car to make money for
  them while not in use.

They give the impression Teslas (with the Full Self Driving package) would get a
[level 5 autonomy][levels]. That is, one could send the car alone to pick up
kids from school and expect it to get them back. The concept of a driver would
cease to exist and the only reason a car would have a steering wheel (as an
extra package) would be to have fun. Something like riding a horse today ‒ one
doesn't acquire that skill to get from point A to point B.

But I don't think it's going to be that way at all. Not at first, not soon.
Tesla is on level 2 right now and expecting it to „jump“ 3 levels at once, or
during a really short time when it's somewhere on level 2 for 10 years is
ambitious.

But I have further reason why I don't believe Tesla or anyone would be getting
level 5 any time now.

*Note:* I'm assuming specific large-scale infrastructure for autonomous cars
isn't going to be built. Not everywhere. Maybe some highways or so, but level 5
means it can drive in however broken, badly marked road a human can.

## What I think the claims *actually* mean

Elon Musk is known to say audacious things and he's good at marketing. He
knows how to stir the waters. I believe these claims aren't outright lies, but
more of an [Aes Sedai truth](https://wot.fandom.com/wiki/Aes_Sedai) ‒
technically true, but somewhat misleading.

But even if I'm right, this would be a big step forward.

I think Tesla is aiming at [level 4][levels]. That is, the car would be driving
and navigating the traffic most of the time, could detect when there's something
it can't handle and request assistance. It could pull over and wait for the
assistance safely.

Such assistance would be more high level ‒ not fine adjusting of direction or
speed. More like picking the right lane or correcting a speed limit. These
can be done on the touch screen and don't really need a steering wheel or
pedals. They still need a driver, even though such driver doesn't have to pay
attention at all times.

### Implementation of robo taxis

Given Teslas are connected to the Internet, such driver wouldn't need to be
physically present inside the car.

If enough people allow their unused cars to be used by such system, there would
be always one close by, no matter how small a town or settlement. A driver
wouldn't have to be waiting inside each car, one would be assigned only once the
car gets used. Some companies seem to be doing something similar as an
experiment in the Bay Area. Tesla just wants to reuse the cars owned by folks.

By the way, I don't have hopes this would be an open market. Tesla would, of
course, be the sole operator of such service. Besides it being a good business,
they could use such taxi driving as further training for the autopilot.

## Why I'm sceptical to a *fully* autonomous cars arriving soon

I'm not going to talk about if Tesla's approach of machine learning from
examples of human drivers is better or worse than hardcoding rules, as seem to
be the case of many other manufacturers. I'm simply going to argue that
*neither* is going to give us level 5 soon because there are two more „cognitive
jumps“ on the way, each one significantly more complex than the previous.

Technology may be speeding up exponentially, but so is the complexity of the
tasks here. I expect one of the jumps to happen soon, but not both. Each one
seems to take about 10 years at least, there's no reason to think the most
complex of them would happen in no time at all. It'll be evolution, not a
revolution.

So, let's go over the steps. These are not based on the levels of self-driving,
they are based on the complexity.

## The past: Simple local systems

These are the assistants we have for longer time. They have some very simple,
yet useful, local function. These are cruise controls, ABS, traction control…

These basically need only a simple feedback loop. With a cruise control, if the
car is going too slow (measured simply by counting how many times the wheels
turn per second), it adds more fuel / power. If it is going too fast, it uses
less fuel / power. That's it. Any coder after the first year of university is
expected to be able to do this in a week. A geek with enough motivation will be
able to do a mechanical version from Lego bricks.

They operate in well-defined conditions. They have few sensors wired to them,
know what to expect and deal only with themselves.

## The present: Making sense of the external world

One step further are things like adaptive cruise control (the one that slows
down behind another car), keeping in lane, emergency braking or avoiding
collisions.

For these to work, the car must build a model of the surroundings. It doesn't
need to particularly _understand_ what these things around are. It's enough to
detect lines on the road, it is enough to detect that there's something big
moving and that if it doesn't speed up by more than 10% in the next second, the
car won't crash into it.

That's significantly more effort to make work than the above. Gathering sensor
data from a lot of sensors, clearing it of noise, normalizing (compensating for
different lighting conditions), putting all of that together and somehow
tracking that the blob of pixels from the picture 20ms ago looks similar to
this blob of pixels from the current picture. Then some more math, probabilities
and physics in some 3D vector space.

It's a lot of work, lot of tasks, but all of them are more or less well
understood. It's done research, it's „only“ engineering. Put enough skilled
people into the room with textbooks and computers and they'll come up with the
result in an expected time frame.

This is basically what gets us to Level 2 ‒ what we have now. An augmented
driver. This is safer and more convenient, because a car:

* Can look full 360° around at once, unlike human with just two eyes pointed
  ahead. It can use other types of sensors (sonar, …) to compensate for bad
  conditions affecting only one type.
* Has significantly faster reflexes than humans and more precise control.
* Doesn't get tired and doesn't fall asleep while driving.
* Doesn't have to start learning from scratch every time a new car is made.
  Every human starts as a beginner.

Still, humans are needed for the higher understanding of the situation ‒
intentions of other drivers, traffic rules, etc.

## The near future: Experience

In this step, the car needs to be able to step cognitively up from executing
simple tasks (keeping in a lane while not crashing into anyone) to complex
decisions. It needs „experience“ to be put inside it somehow. Going from
„something behind me“ to „a police car with flashing blue lights behind me,
which means I need to pull over and let it pass“.

It's no longer just math and physics. This is getting „fuzzier“. Recognizing
signs. Having statistics about the fact that a car flashing a direction light
will at 95% chance go in that direction on the next intersection (the other 5%
is forgotten light).

This is what the car makers are working on. In case of Tesla, it's feeding a
neural net with huge amount of human drivers doing stuff and trying to act the
same. In other cases, it's hardcoding rules. It means teaching the car about
traffic rules, it means teaching it how to pick up the right lane on an
intersection, it means teaching it that a lot of drivers don't follow the
traffic rules.

This is still work in progress and it is an active research. It's on the level
we have an idea how to go about that research and we have a pretty good idea
that it'll more or less work, with some unknown roadblocks on the way that'll
need solving. We know that we'll make it work, we are not sure when.

It's again vastly more complex stuff than the above. I mean, you can teach a
tortoise to follow a drawn line. You'll have really hard time teaching a
tortoise or even a dog complex interactions on a roundabout or the right of way.

Once this works, it'll allow for the car to do some amount of driving and ask
for assistance once it doesn't know how to deal with something. This gives us
level 3 or 4 (going from 3 to 4 is simple from complexity point of view ‒ it
only needs to pull over and park if the driver is not reacting and pulling over
is definitely something it can do at level 3).

The level 3/4 definitions allows for it to work only at certain situation ‒
maybe just on a highway. But conceptually, this is only about „how much“ of this
complexity is utilized. Driving in a city is harder, it needs more training than
driving on a highway. But it is more of the _same kind_ of training. More
examples, more rules, bigger neural nets or bigger database of rules.

Given large enough training set, it can reach a state where a steering wheel is
not needed, or the car can be operated remotely.

Well, given the right legal framework. But that's a different story.

## The far future: Context & improvisation

The above is for dealing with the usual. However, the reality contains a lot of
unusual, broken, unexpected and surprising. For true autonomous driving, the car
needs to learn to improvise and to understand context. Because it'll hit a
situation it has never seen before, was not present in the training set (or not
prevalent enough to make it past the „noise“), is not described by the rule sets
it has.

It isn't enough to act „same as others“, based on „instinct“. It needs to
understand the principles, the „why“.

Let me demonstrate. Imagine a red circle with the number 30 inside it. It's a
speed limit. Unless it isn't:

* Small complication. If it's `30t`, not `30`, it's a weight limit. So, we need
  to know our weight, the weight of the vehicle in each training example. But
  heavy trucks don't often go to places with a weight limit, because they know
  in advance it's there, so it will be rare in the training data.
* Some scoundrel takes a paint and draws eyes to the `0`. That's vandalism, but
  the sign still applies, but this particular version of the sign was never part
  of the training data.
* A pictogram below it means a condition under which it applies.
  - A truck (motorcycle, horse…) means it applies only to these kinds of
    vehicles.
  - If it contains a cloud, it applies when raining (now weather enters into the
    picture).
  - Some are written in human language (damn!) and reference some arcane part of
    common human knowledge (eg. „During workdays“ ‒ which expects to know about
    national holidays of the given country!).
* If the sign is on the back of another car, it applies to that car, not to us.
  - Unless it is a construction car doing its work ‒ then it applies. For human,
    that's a no-brainer. But it's actually a hard problem, recognizing the
    difference between each construction car type doing its job and the same
    type of costruction car just going somewhere.
* If it is part of and advertisement billboard on the road side. It can even
  have a road painted behind it.
* If it's obviously a forgotten sign that was part of construction work that
  terminated yesterday and this one last sign was not cleared up properly.

Eventually, someone will make a sign that's not standardized and expect it to be
understood. Let's say a crocodile inside a red triangle. For a human, that's
obviously a „Beware of crocodiles“ sign (That sign doesn't exist in Europe, but
if some crocodiles run away and start living in a lake, it makes sense. Other
places, like Australia, might have their own version, but that one would look
different ‒ maybe a yellow triangle). Because we know what crocodiles are, we
know why they are dangerous, we deduce not to exit the car at that place. So, if
we have a tire defect, we would probably go a bit further instead of stopping at
that nice beach by the lake to fix it.

Sometimes, the rules are contradicting. There's a dead-end one way street in
Prague. Not kidding. Actual traffic-sign trap.

Some of these weird things will be prevalent enough to make an impact on the
neural net, or someone will enter them into the rule set. But some of them are
so rare they won't. How many examples of someone having a defect near a
crocodile area there are in the training set? None? Even if there was one,
that'd get drowned into irrelevance by other things.

Humans are good at taking their whole knowledge into account, considering the
current context and coming up with *a* solution (not necessarily a perfect one,
but usually acceptable, by projecting the consequences and discarding obviously
bad ones).

Neural nets are not capable of that (at least not today). Rule sets don't work
well for unknown or contradicting situations either. We actually don't have a
tool discovered for this *yet*. It's not just about making a bigger neural net
(unless the neural net somehow can encompass the equivalent of all common human
knowledge). This needs another cognitive type.

Without this, we can't have a true autonomous car. The human needs to be there
somewhere, even if there's a 0.01% chance the car wouldn't cope. The human
doesn't need to be present in the car, and in fact, if the probability is good
enough, Tesla or other automaker could provide the human operator as some kind
of service behind the scene, part of the „Full Self Driving“ package (just hope
it won't happen at place without connection). That is, a car with very good
level 4 capability and connectivity could be marketed and sold as a level 5 car.

This might be solved eventually. But this is of the kind of problem where we
don't even know how to start the research behind it. There's always some time
between researching something and making it production ready. For that reason, I
don't full autonomy in a year or five, not even from Tesla.

[levels]: https://en.m.wikipedia.org/wiki/Self-driving_car#Levels_of_driving_automation
[buy one]: {% post_url 2022-05-06-choosing-ev %}
