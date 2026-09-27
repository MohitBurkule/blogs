# What two EMG sensors taught me about my triceps

### I built a gym app that logs sets and reps from muscle signals. The interesting part wasn't the counting. It was finding out which of my own assumptions about training were wrong, and which of my measurements were.

---

After [getting rid of the dongle](myoblue-without-the-dongle.md), I had two ELEMYO MYOblue EMG sensors
streaming to my phone. The obvious next step was a gym log that fills itself in: strap a sensor on each
triceps, train, and have the phone work out the sets, reps, holds and tempo, without me tapping start and
stop on every set.

That app is [MyoLift](https://github.com/MohitBurkule/myolift). It's an Expo/React Native Android app with
a native Kotlin module for Bluetooth and background recording. It came together over a few weeks of
testing in the gym and fixing what broke. GitHub Actions builds, emulator-tests and releases every
version, and the app updates itself from GitHub releases, so a fix pushed in the afternoon was on my
phone by the evening session.

This post is about what I learned from it: about EMG, about my own body, and about how easy it is to fool
yourself with a signal that looks meaningful.

## What EMG actually measures

EMG is the electrical activity of the muscle under the sensor. It rises when more motor units switch on and
when they fire faster. It is *not* force. The same EMG can mean very different force depending on the
muscle's length, how fast it's moving, whether it's lengthening or shortening, and how tired it is.

That sounds like a technicality. It turned out to be the whole story.

## Counting reps is harder than it looks

The first version counted reps as peaks in the EMG. On my first real session it counted 70 reps where I'd
done about 36. Each of my rope pushdown reps has two humps: a hard squeeze at lockout, then a slow, lower
bump on the way back up. Peak counting saw both.

Fixing that meant requiring a rep to rise well above the surrounding signal (prominence), smoothing, and
finding rep timing from both arms combined. Then the machine triceps pushdown broke it again, and in the
opposite direction: on that machine, activation *drops* at lockout, because the machine's geometry takes
the load when your arms are straight. A full rep there is a dip, not a peak. So each exercise now has its
own counting mode, and the app learns from the counts I correct.

Two things came out of this that I didn't expect:

- **My slow release really is low effort.** On the rope, the way up ran at about 27–34% of the lockout peak.
  I'd suspected it, and the sensors confirmed it.
- **Partial reps look like full reps in EMG.** Half reps at the end of a set produce peaks as tall as full
  ones. EMG alone can't reliably tell them apart.

## Adding a camera

That last point pushed me to add video. The app records the camera and EMG together on the phone's clock,
and I ran a session of deliberate experiments: holds at different positions, partials of each kind, slow
reps, a drop set, one-arm lowering, then assisted pull-ups and dips.

The first surprise was that standard pose detection (MediaPipe) was nearly useless. My phone was close,
often looking down along my arm, and my elbow left the frame at the top of every rep, at which point the
model helpfully placed it at my hands.

What worked instead was not detecting me at all:

- **On the rope,** optical flow on the rope and hands gives their vertical position over time. How far each
  rep travels, compared with the recording's full reps, separates full reps from partials. On my usual set,
  EMG alone called 16 of 25 reps full. The video called 8 full and 17 partial, which matched my notes.
- **On the pull-up and dip machine,** the phone rode on the knee pad looking up, so it moved with me.
  The overhead frame is fixed, so its apparent *size* in the image tells you how close you are to the bar.
  No person detection needed. (This was my idea, and I was pleased it worked.)

Later, Meta's **SAM 3D Body**, which fits a whole 3D body and fills in parts it can't see, did give real
elbow angles on the rope: about 100° at the top and 4° at lockout on the visible arm, with bone lengths
stable within a few percent. It was wrong exactly where you'd expect: when my arms were completely out of
view on the pull-ups, it draped them over my thighs.

## The calibration squeeze was lying to me

The standard way to make EMG comparable is to express it as a percentage of a maximum voluntary
contraction: squeeze as hard as you can for a few seconds, and call that 100%.

My sets kept reading 150–230% of my "maximum". That's not superhuman effort. It means the squeeze was weak,
which is unsurprising: I was doing it cold, at the start of the session, with no warm-up. Every percentage
downstream was inflated, and the fatigue model I'd built broke because of it.

The fix was to stop asking for a maximum at all. Across one session I'd done rope pushdowns at 4.5, 18, 23,
36 and 45 kg, so I could fit EMG against the actual load on my fresh reps, per arm. That curve became the
scale: effort is now expressed as "kg-equivalent", and the calibration squeeze turned out to equal a rep at
about 27 kg. That's one more reason to be suspicious of any single calibration contraction.

## What the numbers said about my body

Treating the whole experiment session as one workout (about 50 minutes, with real rest gaps), a few things
stood out:

- **My triceps are strongest at about 69° of elbow bend** (± 9°), from fitting EMG against load and
  angle over 2,924 video frames. At lockout they can produce less than half that force. That explains
  why holding at lockout barely registered: about 15%, the lowest of any position.
- **Lowering is stronger than pushing,** about 1.3× on my right arm: the classic eccentric advantage.
- **Fatigue built steadily and rest didn't clear it.** The same weight cost about 12% more effort by the end,
  and the EMG frequency marker of muscle-fibre fatigue fell from about 136 to 91 Hz on my right arm. Even
  after a 7-minute break, my first reps still cost about 1.27× the effort per kg. That one's low confidence,
  from only 15 sets.
- **My drop set did what drop sets are meant to.** As the weight fell from 36 to 14 kg, my effort stayed at
  23–35 kg-equivalent: near-maximal drive at every weight.
- **Dips gave my triceps the highest activation of the day.** More than a 45 kg rope pushdown.

## Is any of it actually growing muscle?

This was the question that mattered, and the one the sensors can't answer on their own. EMG shows what
happened in the muscle in that moment. Whether that leads to growth is a question for long-term training
studies. And researchers are clear that acute EMG amplitude is
[not a validated predictor of hypertrophy](https://link.springer.com/article/10.1007/s40279-021-01619-2).

So the app now pairs each technique it detects with what the research says about it, and shows my own
numbers next to each verdict:

- **Drop sets:** helpful. [Same growth in about half to a third of the time](https://link.springer.com/article/10.1186/s40798-023-00620-5),
  as long as each drop goes near failure.
- **My usual tempo** (about 2 s up, 3 s down): helpful. Reps from 0.5 to 8 seconds
  [grow muscle about equally](https://pubmed.ncbi.nlm.nih.gov/25601394/).
- **My "slowest possible" reps** (12.8 s each): less effective. Slower than about 10 s per rep grows less.
- **Partials at the arms-straight end:** less effective. **Partials at the stretched end:**
  [about as good as full reps](https://peerj.com/articles/18904/).
- **Holds:** at the stretch, worth it if you add weight. At lockout, mostly rest.
- **Only doing pushdowns:** add an overhead extension. In one trial it
  [grew the triceps about 40% more](https://www.tandfonline.com/doi/full/10.1080/17461391.2022.2100279).

### The one I got wrong

I'd been doing a trick on the rope: push down with both arms, lower mostly with one. My first analysis
said it worked: the lowering arm carried 2.15× the other's activation. A meta-analysis said extra eccentric
load doesn't add growth, so I filed it as "harder, not better".

Both conclusions were too confident.

When I re-ran the analysis with the lowering phase timed from the video instead of from the EMG itself,
the effect disappeared into my normal left/right imbalance. Two reasonable methods, two different answers.
And when I read the meta-analysis properly, its evidence on muscle *size* came from about three short
trials, mostly using weight releasers with pauses rather than continuous one-arm overload like mine.
"No detectable difference" in underpowered data isn't "no benefit".

So the honest verdict is: unknown, plausible, weakly evidenced. The only way to find out for me is to do
it on one arm for a couple of months, train the other normally, and measure both. The app has a body
measurements log for exactly that.

## What I'd take from all this

- **A signal that looks meaningful can still be measuring the wrong thing.** The calibration squeeze,
  the one-arm result and peak counting all looked fine until I checked them another way.
- **Keep the raw data.** Every analysis here re-ran recordings from weeks earlier with better methods.
  The app never throws away a packet.
- **Measure what the muscle does, not just what the brain sends.** Rep speed, range of motion and angle
  from video tell you about output. EMG tells you about drive. The ratio between them, movement per unit
  of drive, is where fatigue shows up most clearly.
- **Look for a second method before believing a surprising result.** The best findings here survived
  being measured two ways. The ones that didn't were the ones I'd been most excited about.

Next up: a short session designed to fill the gaps. That means a few fast reps at three weights for the
force–velocity curve, sets taken to real failure with fixed rests for the fatigue model, and the phone
propped side-on for clean angles. After that, the one-arm-versus-the-other experiment, which will take
months, not minutes.
