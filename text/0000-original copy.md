- Feature Name: domain_restricted_primitive_integers
- Start Date: 2026-09-24
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

## Summary
[summary]: #summary
Add a new core 'primordial' integer type "DomainInt<S, [D]>" with definable finite intervals for the intent of improving programmer ergonomics and to allow compile time assurance that invalid values can never occur in the type. Redefine existing integer types such as signed/unsigned/NonZero as aliases of this type to unify behavior.

"DomainInt<S, [D]>"
Where 'S' is a enum value representing the number of bits used for the underlying storage type, this may be restricted to only a few valid numbers that the language currently supports such as i/u 8, 16, 32, 64, 128, but could be expanded later if arbitrary or other integer sizes are added.
Where '[D]' is a set of intervals representing all valid values this DomainInt could be. The canonical format for this would be an array of range types organized from lowest to highest with no overlapping intervals or directly adjacent intervals. E.G [1..101] would represent a valid interval from and including 1 up to and including 100. Non-canonical formats are converted to the same canonical format internally, potentially with a compiler or other process warning to complain.
Empty domains [0..0] may exist as a !-like type but will give a compiler warning if defined by the programmer.

DomainInt type variables will receive certain const values at compile time based on what intervals are valid within them:
-::NON_ZERO - Does this domain not contain the integer 0, ![0, 0].
-::NON_NEGATIVE - Does this domain not contain any negative integers, ![-∞, -1].
-::NON_POSITIVE - Does this domain not contain any positive integers outside zero, ![1, ∞].
-::IS_CONTIGUOUS - If this domain is a contiguous set of values([1,20]) and not a disjoint union([1,10]∪[15,20]).

DomainInt's that are subsets (A⊆B) of another DomainInt will receive the [Into] trait. 'B = A.into()' is valid in this scenario.
DomainInt's may be directly assigned literal integers in code, these will error at compile time if the assignment is not valid.
    let degrees: DomainInt<64, [1..361]> = 5; //Ok
    let bad_degrees: DomainInt<64, [1..361]> = 400; //Compile error
Any runtime or undeterminable conversion/declaration must use the Result family of functions where an Error is returned if the tried value is not valid.

The intervals declared must fit within the underlying storage type, if the underlying type is an i64 then the lower and upper limits of the domain could not go past i64::MIN and i64::MAX respectively. If the domain is larger or outside (negative values in a unsigned storage type) valid bounds the type will throw a compiler error.
For the purposes of this RFC the underlying storage type must be able to contain the values as-is, any 'export' of the value to a different domain will be directly converted to that underlying type first. The compiler will also be able to easilly determine niches given that any value within the domain of the underlying type that is not within the domain of the type is essentially free use.

[Contiguous] DomainInt's will retain the existing mathmatical operators, where A+B panics in debug mode and wraps around in release mode. This is to keep compatability with existing integer code. Otherwise they will contain the same existing functions: checked_*, saturating_*, wrapping_*, overflowing_*, etc. 
The MIN and MAX of the domain will affect where values wrap or saturate. [1..361] will saturate to a MAX of 360, and 361 will wrap around to the MIN of 1.

[NonContiguous] DomainInt's can be consumed for mathematical functions but will not allow math to be performed directly on them. Programmers will need to convert the [NonContiguous] Domain to a [Contiguous] one and back to perform math on them.


## Motivation
[motivation]: #motivation

NonZero<T> requiring a constructor to assign a literal value annoyed me.
In game development many values have explicit known ranges. A player attribute may only range from 1 to 20, and a player skill may only range from 1 to 100 and any attempt to go outside those bounds may only need saturating results or should just be invalid. I may want to add attribute to skill but avoid overflowing my bounds. The only way to do this in Rust cleanly is to build my own structs and override the mathematical operators to work on the value stored inside along with the validation mechanic for handling overflows. Alternatively I can just use i64's for everything and hope I remember to check my bounds, losing any value of type protection.
This results in a lot of excessive code for something that is already done arbitrarilly for the pre-existing primitive types. The u64 type just says "I want a domain of 0 to u64::MAX", the NonZeroI8 type just says "I want a domain of i8::MIN to -1 and 1 to i8::MAX".

Adding the hypothetical 'Primordial' type that allows the programmer to directly declare what they consider a valid domain of integers seem likes the most powerful solution to this matter with the most knock-on benefits.

Domains existing at the compiler level means values can be very cleanly prescribed with minimal code.
Pretend you're an expert robot programmer, Consider something like:
```
    struct RotationHolder {
        //Controller must receive values as 32 bits.
        base: Wrapping<DomainInt<i32, [0..360]>>, //Base can rotate freely. 0 is straight forward.
        elbow: Saturating<DomainInt<i32, [0..181]>>, //Elbow can only rotate 90 degrees. 0 is straight out.
        wrist: Saturating<DomainInt<i32, [-135..136]>>, //Wrist can only twist 270 degrees, 0 degrees has the gripper surface parallel with the ground.
    }

    //Startup arrangement, Rotate arm 90 degrees to show startup.
    RotationHolder.base += 90;
    RotationHolder.elbow = 0;
    RotationHolder.wrist = 0;
    submit_commands(RotationHolder.convert_to_api_format())
```
Given this code, you cannot order your imaginary expensive robot to go to an invalid location which would break it and get you fired, and when it does break your replacement can understand it quickly.

Seperating domains from the binary storage value will also open up more downstream refinement options:
-Being able to describe specific smaller domains allows for smaller exaustive checking code where used.
-Functions and libraries that rely on integer arguments can be domain restricted to only valid values allowing for cleaner API's with 'never-fail' functions that do not need to return Option/Result values, and invalid-value checking is pushed to the programmer and enforced by the compiler who will likely have a better understanding of their data and how to optimize it against the function requirements.
-Functions can return known or calculated domains. If a function only ever returns the values 1 to 10 the writer can explictly specify this.
-Potential future path for Domains of enums, enums are stored using integer discriminants, if a function only takes or returns certain enums this would allow specification of that in the function declaration. Allowing a type to only be a specific set of values in an enum has been a requested feature.
-There is a future path for a 'Domain Checker', the resulting domain of mathematically combining two domains bounded by the underlying type can be determined at compile time (risking a combinatorial explosion for very large specific domains). Potentially allowing the compiler to automatically derive potential domains returned from functions and then provide the domain benefits for the rest throughout the entire program.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Explain the proposal as if it was already included in the language and you were teaching it to another Rust programmer. That generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how Rust programmers should *think* about the feature, and how it should impact the way they use Rust. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, describe the differences between teaching this to existing Rust programmers and new Rust programmers.
- Discuss how this impacts the ability to read, understand, and maintain Rust code. Code is read and modified far more often than written; will the proposed feature make code easier to maintain?

For implementation-oriented RFCs (e.g. for compiler internals), this section should focus on how compiler contributors should think about the change, and give examples of its concrete impact. For policy RFCs, this section should provide an example-driven introduction to the policy, and explain its impact in concrete terms.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

## Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?
- If this is a language proposal, could this be done in a library or macro instead? Does the proposed change make Rust code easier or harder to read, understand, and maintain?

## Prior art
[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- For language, library, cargo, tools, and compiler proposals: Does this feature exist in other programming languages and what experience have their community had?
- For community proposals: Is this done by some other community and what were their experiences with it?
- For other teams: What lessons can we learn from what other communities have done here?
- Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other languages, provide readers of your RFC with a fuller picture.
If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other languages.

Note that while precedent set by other languages is some motivation, it does not on its own motivate an RFC.
Please also take into consideration that rust sometimes intentionally diverges from common language features.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

## Future possibilities
[future-possibilities]: #future-possibilities

Think about what the natural extension and evolution of your proposal would
be and how it would affect the language and project as a whole in a holistic
way. Try to use this section as a tool to more fully consider all possible
interactions with the project and language in your proposal.
Also consider how this all fits into the roadmap for the project
and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the
RFC you are writing but otherwise related.

If you have tried and cannot think of any future possibilities,
you may simply state that you cannot think of anything.

Note that having something written down in the future-possibilities section
is not a reason to accept the current or a future RFC; such notes should be
in the section on motivation or rationale in this or subsequent RFCs.
The section merely provides additional information.
