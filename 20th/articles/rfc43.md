## Intro

RFC 43, titled ['Integrate BigInts (and BigRats) Support Tightly With The Basic Scalars'](https://raku.org/archive/rfc/43.html) was submitted by Jarkko Hietaniemi  on 5 August 2000. It remains at version 1 and was never frozen during the official RFC review process.

Despite this somewhat "unoffical" seeming status, the rational (or `Rat`) numeric type, _by default,_ powers all of the fractional mathematics in the [Raku](https://raku.org) programming language today.

You might say that this RFC was "adopted, and then some."

## A legacy of imprecision

There is a dirty secret at the core of computing: most computations of reasonable complexity are imprecise. Specifically, computations involving floating point arithmetic have known imprecision _dynamics_ even while the imprecision _effects_ are largely un-mapped and under-discussed. 

The standard logic is that this imprecision is so small in it's _individual_ expression -- in other words, because the imprecision is so negligible when considered in the context of an individual equation it is taken for granted that the overall imprecision of the system is "fine."

Testing the accuracy of this gut feeling in most , however, would involve re-creating the business logic of that given system to use a different, more accurate representation for fractional values. This is a luxury that most projects do not get.

## Trust the data, question the handwaves

What could be much more worrisome, however, is the failure rate of systems that _do_ attempt the conversion. In researching this topic I almost immediately encountered third party anecdotes. Both come from a single individual and arrived withing minutes of broaching the floating point imprecision question.[^1]

One related the unresolved issue of a company that cannot switch their accounting from floating point to integer math -- a rewrite that must necessarily touch every equation in the system -- without "losing" money. Since it would "cost" the company too much to compute their own accounting accurately, they simply don't.

Another anecdote related the story of a health equipment vendor whose equipment became less effective at curing cancer when they attempted to migrate from floating point to arbitrary precision.

In a stunning example of an exception that just might prove the rule of "it's a feature, not a bug", it turned out that the tightening up of the equation ingredients resulted in a less effective dose of radiation _because the floating point was producing systemic imprecisions that made a device output radiation beyond it's designed specification._

In both cases it could be argued that the better way to do things would be to re-work the systems so that expectations matched reality. I do have much more sympathy for continued use of the life-saving imprecision than I do for a company's management to prefer living in a dream world of made up money than actually accounting for themselves in reality, though.

In fact, many financing and accounting departments have strict policies banning floating point arithmetic. Should we really only be so careful when money is on the line?

## Precision-first programming

It is probably safe to say that when Jarkko submitted his RFC for native support high-precision `bigint`/`bigrat` types that he didn't necessarily imagine that his proposal might result in the adoption of "non-big" rats (that it is, arbitrary precision rationals shortened into typically playful Perl-ese) as the default fractional representation in Perl 6.[^2]

Quoting the thrust of Jarkko's proposal, with emphasis added:

> Currently Perl 'transparently' starts using double floating point numbers when the numeric values grow too large for the native integer types (int, long, quad) can no more hold quantities that large. Because double floats are at their heart a lie, they cannot truly represent large numbers accurately. Therefore **_sometimes when the application would prefer to stay accurate,_** the use of 'bigints' (and for division, 'bigrats') would be preferable.

Larry, it seems, decided to focus on a different phrase in the stated problem: _"because double floats are at their heart a lie"._ In a radical break from the dominant performance-first paradigm, it was decided that the core representation of fractional values would default to the most precise available -- regardless of the performance implications.

Perl has always had a focus on "losing as little information as possible" when it comes to your data. Scalars dynamically change shape and type based on what you put into them at any given time in order to ensure that they can hold that new value.

Perl has lso always had a focus on DWIM -- Do What I Mean. Combining DWIM with the "lose approximately nothing" principle in the case of division of numbers, Perl 6 would thus default to understanding your meaning to be a desire to have a precise fractional representation of this math that you just asked the computer to compute.

Likewise, Perl has also always had a focus on putting in curbs where other languages build walls. Nothing would force the user to perform their math at the "speed of rational" (to turn a new phrase) as they would have the still-essential `Num` type available.

In this sense, nothing was removed -- rather, `Rat` was added and the default behavior of representing parsed decimal values was to create a `Rat` instead of a `Num`. Many languages introduce rationals as a separate syntax, thus making precision opt-in (a la `1r5` in J). In Perl 6 (and eventually thus Raku), the opposite is true -- those who want imprecision must opt-in instead.

## A visual example thanks to a matrix named Hilbert

In pursuit of a nice example of programming concerns solved by arbitrary precision rationals  that are a bit less abstract than the naturally unfathomable "we have no _actual_ idea how large the problem of imprecision is in effect on our individual systems let alone society as a whole", I came across the [excellent presentation](https://www.youtube.com/watch?v=CkaQQYcxpfM) from 2011 by Roger Hui of Dyalog where he demonstrates a development version of Dyalog APL which included rational numbers.

In his presentation he uses the example of Hilbert matrices, a very simple algorithm for generating a matrix of any size that will be of notorious difficulty for getting a "clean" identity matrix (if that's not clear at the moment, don't worry, the upcoming visual examples should make this clear enough to us non-experts in matrix algebra).

Here is the (very) procedural for generating our Hilberts for comparison (full script in [this gist](https://gist.github.com/ab5tract/3e25e4a2ce63a349b7eb4601a85b6993#file-rationale-matrique-raku)):[^3]

```
my %TYPES = :Num(1e0), :Rat(1);
subset FractionalRepresentation of Str where {%TYPES{$^t}:exists};
sub generate-hilbert($n, $type) {
    my @hilbert = [ [] xx $n ];
    for 1..$n -> $i {
        for 1..$n -> $j {
            @hilbert[$i-1;$j-1] = %TYPES{$type} / ($i + $j - 1);
        }
    }
    @hilbert
}
```

One of the most important aspects of having rationals as a first-class member of your numeric type hierarchy is that only extremely minimal changes are required of the math to switch between rational and floating point.

There is a danger, as Roger Hui notes in both his video and in a follow-up email to a query I sent about "where the rationals" went, that rational math will seep out into your application and unintentionally slow everything down. This is a valid concern that I will return to in just a bit.


### Floating Hilbert

Here are the results of the floating point dot product between a Hilbert and it's inverse -- an operation that generally reseults in an identity matrix (all 0's except for a straight diagonal of 1's from the top left corner down to the bottom right).

```
Floating Hilbert
         1       0.5  0.333333      0.25       0.2
       0.5  0.333333      0.25       0.2  0.166667
  0.333333      0.25       0.2  0.166667  0.142857
      0.25       0.2  0.166667  0.142857     0.125
       0.2  0.166667  0.142857     0.125  0.111111
 
Inverse of Floating Hilbert
     25    -300     1050    -1400     630
   -300    4800   -18900    26880  -12600
   1050  -18900    79380  -117600   56700
  -1400   26880  -117600   179200  -88200
    630  -12600    56700   -88200   44100

Floating Hilbert ⋅ Inverse of Floating Hilbert
            1             0            0             0            0
            0             1            0  -7.27596e-12  1.81899e-12
  2.84217e-14  -6.82121e-13            1  -3.63798e-12            0
  1.42109e-14  -2.27374e-13  2.72848e-12             1            0
            0  -2.27374e-13  9.09495e-13  -1.81899e-12            1

```

All those tiny floating point values are "infinitisemal" -- yet some human needs to choose at what point of precision we determine the cutoff. Since we "know" that the inverse dot product is supposed to yield an identity matrix for Hilberts, we can code our algorithm to translate everything below `e-11` into zeroes.

But what about situations that aren't so certain? Some programmers undoubtedly write formal proofs of the safety of using a given cutoff -- however I doubt any claims that this population represents a significant subset of programmers based on lived experience.

### Rational Hilbert

With the rational representation of Hilbert, it's a lot easier to see what is going on with the Hilbert algorithm as the pattern in the rationals clearly evokes the progression. This delivers an ability to "reason backwards" about the numeric data in a way that is not quite as seamless with decimal notation of fractions.[^4]

```
Rational Hilbert
    1  1/2  1/3  1/4  1/5
  1/2  1/3  1/4  1/5  1/6
  1/3  1/4  1/5  1/6  1/7
  1/4  1/5  1/6  1/7  1/8
  1/5  1/6  1/7  1/8  1/9

Inverse (same output, trimmed here for space)

Rational Hilbert ⋅ Inverse of Rational Hilbert
  1  0  0  0  0
  0  1  0  0  0
  0  0  1  0  0
  0  0  0  1  0
  0  0  0  0  1
```

Notice the lack of ambiguity in both the initial Hilbert data set and the final output of the inverse dot product. There is no need to choose any threshold values, thus no programmer decisions come in between the result of the equation provided by the computer and the result of the equation as stored by the computer or presented to the user.

Consider again the fact that it is useless to do an equality comparison in floating point math -- the imprecision of the underlying representation forces users to choose some decimal place to end the number they check against. There is no possible model that can take into account the total degree of impact this human interaction (in aggregate) creates on top of the known and quantifiable imprecisions that are usually the basis of the "it's good enough" argument.

## The cost of precision

In summary, the nice equations about maximumum loss per equation are far from the entire story and it is in recognition of that fact that Raku does it's best to ensure that the eventual user of whatever Raku-powered system is not adversely impacted by unintentional or negligent errors in the underlying math used by the programmer.

It nevertheless remains a controversial decision that has a significant impact on baseline Raku performance. Roger Hui's warning about rational math "leaking out" into the business application is well-served by the timing results I get for the current implementations in `Matrix::Math`:

```
raku rationale-matrix.p6 --timings-only 8
Created 'Num' Hilbert of 8x8: 0.03109376s
Created 'Rat' Hilbert of 8x8: 0.0045802s
Starting 'Num' Hilbert test at 2020-08-10T08:38:20.147506+02:00
'Num' Hilbert inverse calculation: 4.6506456s
'Num' dot product: 4.6612038s
Starting 'Rat' Hilbert test at 2020-08-10T08:38:24.808709+02:00
'Rat' Hilbert inverse calculation: 5.0791889s
'Rat' dot product: 5.0791889s
```

Considering it is quite unlikely for the floating point version to be so close to the speed of the rational one, this simple benchmark appears to prove the case for Roger's warning about rational math "leaking out" into the larger system and causing significant performance degradation.[^5]

So even though we ourselves took the extra step to try and get floating point (im-)precision and thus speeds, we were thwarted in our attempts. I believe this is a valid concern for the plenty of use cases that benefit from floating point math without causing any harm.

## Potential futures of even more rational and precise rationality

The new capababilities provided by the upcoming introduction of Raku AST include the ability to modify/extend the behavior of dispatch. It then becomes quite feasible for a user-space module to be able to tweak the behavior of core to the extent that a more forceful approach to using floating point representation could be applied.

In the current preliminary conceptual phase, the idea would be to provide means to test the necessity of rational representation to the outcome of a system. This could be achieved by making the underlying numeric type "promotion" logic configurable. It then becomes possible to imagine replacing the `Rat` type with a `Bell` representation that can track the overall loss of precision throughout the system.

If that is below a given threshold, the same configurability could be used to remove `Rat` from the hierarchy entirely, thus ensuring floating point performance.

## Balancing concerns

The concerns of unintentional and/or unrecoverable performance degradation remain extremely valid and up until beginning the research on this I was un-concerned about rational performance.

This lack of concern relied on a huge caveat -- that floating point performance is available and guarantee-able to anyone who wants to ensure it's use throughout their system.

Unfortunately this is not the case in current Rakudo.

I am still likely to move forward with a requirement of provable precision as a personal requirement when judging a _system_ for whether it meets my  expectations for "21st century."

However, I now disavow my previous feelings that rational math as a default fractional representation is the best -- or, to put it more playfully, most _rational_ -- route for a given language implementation to take.[^6]

## Conclusion

The intention of defaulting to rational representation was not intended to force users to the performance floor of rationals and it is a natural extension of Perlish philosophy to strive towards being even better at giving the user exactly -- without surprises -- what they want.

Rather, Raku chooses a precision-first philosophy because it fits into the overall philosophy of keeping the truest representation of a user's demands as possible regardless of current computation cost -- without locking them into the expectations of the language implementor either.

To the extent that Raku does not yet get this quite perfect, there remains a certain quality of rebellion and original thinking on this topic specifically and throught the language that puts Raku quite close to providing powerful yet seamless mechanisms for choosing the best fractional representation for any given system.



[^1]: I say this because it implies that these are hardly the only two occasions where people have chosen an imprecise representation, have seen that imprecision manifest at significant scale, and have chosen to continue with the imprecise representation for mostly dubious reasons.
[^2]: To date I'm not aware of any other programming languages designed and implemented in the 21st century that take this same "precision-first" approach. It apparently remains a controversial/unpopular choice in language design.
[^3]: 
  This is called from inside some glue that sepearates the timings and type cheacking from the underlying details.

  ```
  sub inner-hilbert(Int $n, FractionalRepresentation $type, $show-timings) {
      my $start = DateTime.now;
      my $hilbert = Math::Matrix.new: generate-hilbert($n,$type);
    say "Created '$type' Hilbert of {$n}x{$n}: {DateTime.now - $start}s"
          if $show-timings;
      $hilbert
  }
  ```
[^4]: I patched a local version of `Math::Matrix` to print it's rationals always as fractions, so your output will look different for the rational Hilbert (ie the same as fractional Hilbert) if you end up running the code in this [gist](https://gist.github.com/ab5tract/3e25e4a2ce63a349b7eb4601a85b6993#file-rationale-matrique-raku).

[^5]: 
  If the timings of those calculations depress you, it is worth pointing out that Raku developer lichtkind has developed a version of the library that produces timings like these instead.

  ```
  raku rationale-matrix.p6 --timings-only 8
  Created 'Num' Hilbert of 8x8: 0s
  Created 'Rat' Hilbert of 8x8: 0.006514s
  Starting 'Num' Hilbert test at 2020-08-10T11:06:34.567898+02:00
  'Num' Hilbert inverse calculation: 0.2784531s
  'Num' dot product: 0.2784531s
  Starting 'Rat' Hilbert test at 2020-08-10T11:06:34.846350+02:00
  'Rat' Hilbert inverse calculation: 0.06900818s
  'Rat' dot product: 0.0846446s
  ```
  It is likely that this work will be incorporated in the default release at some point. Both systems work much better but here the asymmetry of `Num` to `Rat` performance is looking even more suspicious than the near-parity of the slower, released version. I will do a deeper dive into profiling both versions of the module in a post on my personal blog at some point soon.

[^6]: I want to extend a deep and heartfelt thanks to Roger Hui, dzaima, ngn, Marshall, and the whole [APL Orchard](https://chat.stackexchange.com/rooms/52405/the-apl-orchard) for helping me to arrive at a much deeper understanding of the varieties of class and proportion that are countervailing forces for any hard-line stance relating to rational arithmetic. My positions were worded poorly and argued brusquely and not a match faithful match for your patience. I hope it is some consolation, at least, that I no longer agree with them.
