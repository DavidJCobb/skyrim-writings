
Ever since Skyrim Special Edition's release and with it, the release of several 
improperly  ported Skyrim Classic mods, the community has paid close  attention 
to form  versions, the  version numbers  embedded into  every record in  an ESP 
file. It's widely understood that version 43 is specific to Classic and heralds 
Doom and Death, while version 44 is  specific to Special and is Right and Pure. 
There are  xEdit scripts for scanning your load order in search of  any version 
43 content, and many  authors field  complaints from  users about such content. 
Yet  scattered  findings  and observations  around the  community paint a  more 
complicated  picture,  and suggest  that  version 43 content  isn't  inherently 
dangerous.

This document  aims to bring these observations together and present  a clearer 
picture  of  form  versioning.  My goal  was  to  combat  misinformation  which 
contradicts what is known through reverse-engineering, while also incorporating 
observations made in other scenes  within the Bethesda RPG modding community by 
people with other  specialties. During the process of  writing this document, I 
managed to personally verify some community observations via disassembly, while 
also clarifying  the scope  and scale of other  phenomena I had  noticed in the 
past. It's  my hope that this document can be as educational for others  as the 
process of writing it was for me.

## The short version

* Version 43 data is only harmful in  specific situations. It's not radioactive 
  in the way that much of the community seems to believe.

* Version 43 data is only different from version 44 data in places where Skyrim 
  Special Edition  actually adds new features, and in a small  number of places 
  where Bethesda made some very specific mistakes. Bethesda worked hard to make 
  sure that  Skyrim Special would properly handle Skyrim Classic data  in every 
  case where Bethesda wasn't actually adding new features or functionality.

* Even official game files for Skyrim Special Edition contain a mixture of data 
  in versions 43  and 44. Forms in official files are read  in exactly the same
  way as forms  in modded files, so what works in the former  will also work in 
  the latter.

* Re-saving a  file in the  Creation  Kit is  the recommended way  to port mods 
  between Skyrim Classic and Skyrim Special.

* However, without community fixes,  re-saving a file in the Creation Kit won't 
  properly convert *all* data, because  Bethesda made some mistakes in specific 
  places.

* There are xEdit scripts out there that  can scan your load order and find all 
  mods that contain version 43 data. As  of this writing, these scripts are not 
  advanced enough to identify when that data is actually *out of place.* Do not 
  assume that everything found by these scripts is broken.


## What are the basic differences between form versions 43 and 44?

The main differences between form versions 43 and 44 are:

* Skyrim  Special Edition  allows certain  form types to  specify  new data, to 
  control new features.

* Bethesda went out of their way to try and maintain backward-compatibility for 
  most data, but they made one or two specific mistakes.

This means that  data for version 43  is identical to data for  version 44 when 
all of the following are true:

* The form type  in question  has not  had its  format deliberately  changed by 
  Bethesda.

* Bethesda did  not make mistakes when writing the code to load  the form type.

And indeed,  if you check  the official game files  &mdash;  Dragonborn.esm and 
such &mdash; you'll see both version 43 and version 44 forms inside. Though the 
publicly-available  Creation Kit appears to upgrade the  version numbers on all 
forms when re-saving content,  it seems that  during development, Bethesda only 
changed the version numbers on forms whose structure or content actually needed 
changes.

We'll go  over Bethesda's efforts to  maintain backward-compatibility,  and the 
few mistakes they made in doing so, in a later section.

## What are the differences between forms in official content and forms in mods?

There are none. Some mod authors have  claimed that Skyrim Special Edition uses 
special behavior for the official files only, so that those files can have data 
in version 43 but mods cannot. This is  not true. Forms in official content are 
loaded the same way as forms in modded content.

There are a few  reverse-engineers in the community who have  analyzed Skyrim's 
compiled code in order to determine how it works &mdash; and how to tamper with 
it, to make mods that can't be made with the Creation Kit and Papyrus scripting 
alone. I happen to be one of them. I'm currently working on making my own tools 
for editing mods,  and as part of that  process, I've been  reverse-engineering 
how the game loads forms. My focus is  on Skyrim Classic, but I've occasionally 
cross-referenced against Skyrim Special  itself and against the findings of the 
reverse-engineers who specialize in Skyrim Special.

The basic process that the game has for loading forms is as follows:

* Check if a form  with this ID already exists: are we loading  an entirely new 
  form, or an override?  If it's a new  form, then we need  to create a form in 
  memory to hold the data; if it's an override, however, then we already have a 
  form in memory, so we're going to ask  that form to clear out all of the data 
  it's loaded before and reset itself.

* Once we have a form in memory, with some nice, blank data, we can then call a 
  function on that form to load the data &mdash; a <dfn>virtual function</dfn>, 
  which is just computing jargon for, "We ask the form to do something, and it, 
  not us, decides  how it's gonna do the  thing." There are  over a hundred and 
  twenty different kinds of forms in Skyrim, and each of them knows how to load 
  itself.

There are a few extra details, for things like the Creation Kit's collaboration 
features (which are  relevant to the game as well, because  the Creation Kit is 
basically just  a heavily hacked copy of the game engine). There  are also some 
special cases for things like game  settings (GMST) and NavMeshInfoMaps (NAVI), 
where they either  aren't "real" forms or are otherwise handled  in very unique 
ways. The vast majority of forms follow the two basic steps above, though.

Now, let's dive into some light technical details.

I mentioned  that forms load  their data using a  "virtual function."  The game 
doesn't use the exact same code to load  every form, because the forms all have 
completely different data, often arranged in completely different ways. Rather, 
every form type has a list of functions  attached to it, and the game refers to 
these by number.

The seventh function in a form's virtual function list is used to load a form's 
data from an ESP, ESM, or ESL file. Every  form type has its own such function, 
and the program is  organized so that the code for loading  a form doesn't have 
to ever bother to check what kind of  form it's working with. It knows that for 
*whatever*  form it has,  there will be a function list,  and it  just needs to 
grab the seventh function in that list.  When reverse-engineers like me look at 
Skyrim's code, we can actually find these  lists, study the functions inside of 
the list, and work out what each function is supposed to do.

<!-- NOTE: Computers start from zero, so it's actually virtual function 06. -->

Skyrim's  reverse-engineering  community is fairly  good about  documenting our 
findings, so those lists are publicly visible in places like [CommonLibSSE's source code](https://github.com/Ryan-rsm-McKenzie/CommonLibSSE/blob/master/include/RE/T/TESForm.h#L115).
As of this writing, the `TESForm` class, used as a base for all form types, has 
in Skyrim Special  and in Skyrim Classic the same number of  virtual functions, 
and all  functions that  have been identified  across both  games are  an exact 
match. There isn't some special set of functions used specifically for official 
content only.

This means that if a given piece of  data can safely use version 43 in official 
files, then it  can safely use version 43 in mods. You might see  this in cases 
where  someone uses SSEEdit to create a  Skyrim Special mod from scratch,  with 
overrides of forms that were version 43 in the base game or bundled DLCs.

## Bethesda made some mistakes, so the Creation Kit won't get everything right

The correct way to  port a Skyrim Classic  mod to Skyrim  Special Edition is to 
re-save that  mod in the Skyrim Special Edition Creation Kit.  However, not all 
of the mod's data  will be properly  preserved: there are  specific cases where 
Bethesda made mistakes  while trying to  implement  backward-compatibility with 
Skyrim  Classic,  resulting in  unintentional changes to  the file  format; and 
because these file format  changes are unintentional,  the Creation Kit doesn't 
handle them entirely correctly.

One of the known issues is with critical  hit data on weapons. Every weapon can 
optionally specify  a spell that should be applied to an enemy  when the weapon 
is used to land a critical hit on that  enemy. (This behavior can optionally be 
limited to *fatal* critical hits as well.) However, Bethesda failed to maintain 
backward-compatibility for the critical-hit  data structure (`WEAP/CRDT` in the 
file format), and so in practice the specified effect is lost. In order to gain 
an understanding as to why, let's look at how this data is stored.

We're going to get a bit technical here, so feel free to skip a few headings if 
that's not your scene. Ctrl + F for "What goes wrong when we re-save a weapon?"

### What are structs?

In programming terms,  a "data structure" is a collection  of individual values 
that have been grouped together, generally  because they're part of some whole. 
In C++, the  programming  language that Skyrim is  written in, we use  the term 
`struct` for smaller, simpler data  structures. Each value in a struct is often 
called a "field" or a "member," and each field has a type and a name.

The data structure for a weapon's critical hit data looks like this:

```c++
struct CriticalData {
   uint16_t damage;
   float    multiplier;
   uint8_t  flags;
   FormPtr  effect;
};
```

Each field is  listed as its type, and then its name, with the  name being used 
to actually work with the field when writing program code. The fields inside of 
the critical-hit data, then, are:

<dl>
   <dt>damage</dt>
      <dd>
         The amount  of critical  hit damage that  the weapon deals.  This is a 
         16-bit unsigned  integer &mdash;  that is, a two-byte  value that only 
         holds non-negative whole numbers.
      </dd>
   <dt>multiplier</dt>
      <dd>
         A multiplier used to influence how likely an actor is to land critical 
         hits using  this weapon. This is a floating-point number  &mdash; that 
         is, four bytes used to encode a  decimal number, such as 0.3 or -50.0.
      </dd>
   <dt>flags</dt>
      <dd>
         A set of on-or-off settings which  apply to the weapon. These settings 
         are grouped into an 8-bit number,  with one bit per setting. There are 
         eight bits in a byte, so this is essentially a one-byte number.
      </dd>
   <dt>effect</dt>
      <dd>
         The spell to  apply to any actor that  takes a critical  hit from this 
         weapon. We'll talk about how this is stored later.
      </dd>
</dl>

Each of those fields has a unique type suited to a different purpose. There are 
integer types with different sizes, because if you only plan on storing a small 
number, then you don't need to take up all the space needed for a large number; 
there are  also decimal number types,  which are slower for a CPU  to work with 
(not on any human-noticeable timescale, but if you overuse them, it'll add up). 
A program has  to know what a value's  type is &mdash; how the value  is stored 
&mdash; so that the program knows how to actually read and modify the value.

So now that we know what a struct is,  and how the critical hit data is stored, 
we can begin to talk about what goes wrong with it.


### How do forms refer to each other in a Bethesda RPG?

An experienced Skyrim mod author or mod user will know that every piece of data 
in the game  is a "form," and every form has a unique four-byte  numeric ID. If 
you're familiar with programming, you'll  also know what a "pointer" is; if you 
aren't, let me catch you up.

When a computer  program is running, every piece of data in  that program has a 
<dfn>memory address</dfn>. This is basically a unique number that describes the 
location of  that data, kind of  like how a street  address is a  unique number 
that describes the  location of a house. When you have two  pieces of data in a 
computer's memory, and one of them needs to refer to the other, it will usually 
do that by storing the other piece of data's memory address. Your program needs 
to know the  memory address of a piece of data in order to work  with it, so if 
you hold onto something's address, you  can access it at any time. We call this 
held-onto address a "pointer." It "points" to some other object.

When you have two forms in memory, then, the most efficient way for one form to 
refer to the other is by storing a  pointer to the other form. However, this is 
only possible  when both forms actually *are* in memory. If  something isn't in 
memory, then it's not going to have a  memory address &mdash; it's not going to 
have a location in memory. That leads to  an interesting question: how do forms 
refer to each other *while* they're being loaded? They use form IDs inside of a 
file, and they  use pointers inside of memory, but what do they  use when we're 
actively loading them *from* a file *into* memory?

In programming, we can create what's called a <dfn>union type</dfn>. If a value 
has a union  type, that means that it  can be one of several  different things, 
depending on what the situation  calls for. Unions are very similar to structs, 
so let's take a look at a basic example:

```c++
union ExampleUnion {
   uint64_t integer;
   float    decimal;
};
```

If we create  a variable whose type is  `ExampleUnion`, then  that variable can 
hold *either of* the following values, but not both:

<dl>
   <dt>integer</dt>
      <dd>
         An eight-byte  non-negative integer.  This can store  any whole number 
         between 0 and 18,446,744,073,709,551,615.
      </dd>
   <dt>decimal</dt>
      <dd>
         A four-byte  floating-point number. This can store  decimal numbers as 
         massive as 3e38 (a three followed by thirty-eight zeroes), or as small 
         and precise as  1.1e-38. However,  the further  you get from zero, the 
         less precise the storage gets. They're also slower to work with if you 
         only need a whole number.
      </dd>
</dl>

A variable whose type is `ExampleUnion`  can store an integer *or* it can store 
a decimal number, but it cannot store both. Moreover, it needs to have room for 
all of  the things it  can store. If the  largest thing  it can store  is eight 
bytes long,  then the union  needs to  have eight  bytes to work  with, even if 
right now it's  just storing  something that's four bytes.  That means that our 
`ExampleUnion` above takes up eight bytes,  because it's supposed to be able to 
hold an eight-byte integer.

Bethesda has a really clever union type in their code, which I called `FormPtr` 
above. It's a union of a pointer and a form ID. The basic idea is that Bethesda 
loads an ESP file (or ESM or ESL) in two steps. In the first step, Skyrim loads 
all of the forms into memory, but in the places where those forms would usually 
have a pointer, Skyrim instead just stores  the form IDs that it pulls from the 
file. Then, in the second step of the process, when all of the forms are loaded 
into memory and it's possible to create pointers to them, Skyrim goes back over 
all of the forms that it just loaded, swapping out those form IDs for pointers.

### Okay, so what's the catch?

On a 32-bit system, a pointer is four bytes &mdash; the same number of bytes as 
a form ID. On a  64-bit system, however,  a pointer is eight  bytes. This means 
that every `FormPtr` needs to be twice  as long in Skyrim Special Edition. Now, 
normally, that  wouldn't be an issue.  After all, we're just  talking about how 
things are arranged in Skyrim's memory, and that doesn't necessarily have to be 
how things are arranged in an ESP file.

The thing is,  Skyrim Classic often loads structs by copying them  directly out 
of a file and into memory... which means  that these structs' values have to be 
arranged the same way in a file as they are in memory. Skyrim Special *usually* 
accounts for  this by loading the structs  one field at a time,  but there's at 
least one specific  place where  Bethesda actually forgot  to do that: critical 
hit data for weapons.  Skyrim Special and its Creation Kit  *also* blindly copy 
that data into and out of files. This  means that the way critical hit data for 
weapons has to be arranged will actually change depending on whether it's being 
loaded by Skyrim Classic or Skyrim Special.

The most obvious consequence this has  is that the form ID for the critical hit 
effect now needs four bytes of padding in Skyrim Special, because the `FormPtr` 
union is now four bytes larger. However, that form ID is at the end of the data 
structure,  so there actually  isn't much  of a severe  consequence  there; the 
padding is missing, but the game seems  to be able to cope with that just fine. 
There's another wrinkle, though: <dfn>data structure alignment</dfn>.

Let's look back at our struct:

```c++
struct CriticalData {
   uint16_t damage;
   float    multiplier;
   uint8_t  flags;
   FormPtr  effect;
};
```

You might expect each of those fields to  appear in the data one after another. 
However, that's actually not as efficient as it would seem. Skyrim can actually 
work with this struct more efficiently if it's willing to waste a little space.

Remember what I  said about memory addresses? Well, everything  in a computer's 
memory has  a memory address:  if it's in memory, then  it has a  *location* in 
memory. That  applies to forms, but it  also applies to structs  &mdash; and to 
the individual fields in a struct. Fields in a struct also have an <dfn>offset</dfn>, or 
the "distance"  from their  location to  the struct's  location. In  our struct 
above, the `damage`  field has an offset  of 0: it's right at  the start of the 
struct, so there's no distance between it and the start of the struct. Now, the 
`damage` field is two bytes long, so you'd  probably expect the next field, the 
`multiplier` field, to have an offset of 2.

But there's a catch.

Modern CPUs are able to work with an individual value much more quickly if that 
value is <dfn>aligned</dfn>.  This means that its offset  must be a multiple of 
its size.  A two-byte value  must have an offset  that is a  multiple of two; a 
four-byte value  needs an offset that's a  multiple of four; and  an eight-byte 
value needs an offset that is a multiple  of eight. If a field doesn't have the 
right offset, then  it won't be aligned, which means it'll be  slower for a CPU 
to work with.

Modern compilers  will make  sure that every field  in a struct  is aligned, by 
inserting filler  between fields whenever that filler is necessary.  Let's look 
at our  struct, but  this time, I've  indicated where  the filler bytes are for 
Skyrim Classic, and I've also annotated each field with its offset:

```c++
struct CriticalData {
   uint16_t damage;     //  0 // two-byte value
   uint8_t  PADDING;    //  2 // padding byte, to align the next field
   uint8_t  PADDING;    //  3 // padding byte, to align the next field
   float    multiplier; //  4 // four-byte value
   uint8_t  flags;      //  8
   uint8_t  PADDING;    //  9 // padding byte, to align the next field
   uint8_t  PADDING;    // 10 // padding byte, to align the next field
   uint8_t  PADDING;    // 11 // padding byte, to align the next field
   FormPtr  effect;     // 12 // four-byte value
   // End.              // 16
};
```

In Skyrim Classic,  the `FormPtr` &mdash; the union of  a pointer and a form ID 
&mdash; is four bytes,  because pointers and form IDs are  both four bytes, and 
the union  needs room for the largest of  all the values it's allowed  to hold. 
Because  the union is  four bytes  long, it needs  to have  an offset  that's a 
multiple of four, and indeed, we can see  that the padding bytes above are used 
to give it the right offset.

In Skyrim Special, pointers are eight  bytes long, which means that a `FormPtr` 
also needs  to be eight bytes  long, and it needs  to have an offset  that's a 
multiple of eight. That means that we need to include four more padding bytes. 
Here's what the `CriticalData` struct looks like in Skyrim Special:

```c++
struct CriticalData {
   uint16_t damage;     //  0 // two-byte value
   uint8_t  PADDING;    //  2 // padding byte, to align the next field
   uint8_t  PADDING;    //  3 // padding byte, to align the next field
   float    multiplier; //  4 // four-byte value
   uint8_t  flags;      //  8
   uint8_t  PADDING;    //  9 // padding byte, to align the next field
   uint8_t  PADDING;    // 10 // padding byte, to align the next field
   uint8_t  PADDING;    // 11 // padding byte, to align the next field
   uint8_t  PADDING;    // 12 // padding byte, to align the next field
   uint8_t  PADDING;    // 13 // padding byte, to align the next field
   uint8_t  PADDING;    // 14 // padding byte, to align the next field
   uint8_t  PADDING;    // 15 // padding byte, to align the next field
   FormPtr  effect;     // 16 // eight-byte value
   // End.              // 24
};
```

We see that `effect` is now later in the data.

### What goes wrong when we re-save a weapon?

The problems begin when the Skyrim Special Edition Creation Kit first reads our 
weapon and its  critical data. Bethesda didn't *intentionally*  change the file 
format; rather, they forgot to make Skyrim Special Edition and its Creation Kit 
load the  data field-by-field  in order to maintain  compatibility  with Skyrim 
Classic. The change is accidental, so it's not handled at all.

So what happens is, the Creation Kit misreads our data. It thinks that the form 
ID for the critical effect is supposed  to be four bytes later than it actually 
is, so it  takes the form ID that we  provide and it mistakes that  form ID for 
irrelevant padding. Our data structure  ends early, so the Creation Kit ends up 
treating our weapon as if we didn't  specify a form ID, so there is no critical 
hit effect on our weapon anymore.

If you re-save the file in the Creation  Kit, you'll end up with valid data, in 
the sense that  the data will be adjusted to conform to the version  44 format; 
but that adjustment  occurs after the  data has already been  misinterpreted by 
the Creation Kit. It will  dutifully write form  ID zero, the "none" ID, to our 
weapon,  and if (if!) the form ID we  intended is preserved at all,  it will be 
lost in the padding.  The final data is in the right format,  but it's not what 
the mod author intended.

Now, CK Fixes by Nukem  does fix the specific issue with  weapons' critical hit 
data; it  patches the Creation  Kit to do a proper  conversion,  and I strongly 
encourage installing it for that reason  and others. It's important to be aware 
of the limitations  of the tools we use &mdash; and of the  fixes available for 
them  &mdash;  and  one of those  limitations is  that  the  developer-intended 
workflow for  porting mods will only work when dealing with changes  and issues 
that the developer was actually aware of.

## Conclusions

Ultimately,  version 43  data is not  radioactive in  the way that  much of the 
community believes. There are many  situations where it is definitively safe to 
have version  43 data in files intended for use in Skyrim Special  Edition, and 
even when version 43 data is incorrect  and therefore corrupt, that data is not 
necessarily  guaranteed to  obliterate  someone's  game or  save files  (though 
obviously, I  strongly encourage mod authors to avoid shipping  corrupt data to 
their users, and I will  never say that  it's *safe* to  supply corrupt data to 
the game).

It's common for mod authors and users alike to use scripts that are intended to 
scan a load order and identify any mods  that contain any version 43 data; it's 
fortunately somewhat  less common, but still too common, for  users to react to 
this by immediately contacting the authors  of these mods and telling them that 
they must resave the mod in the Creation Kit in order to avoid certain disaster 
and keep from bringing blight upon the land. This is excessive.

If I must give concrete recommendations, they would be...

### For authors

* If you're  porting a mod from Skyrim  Classic to Skyrim Special,  please just 
  re-save it in the Creation Kit. Make  sure to install Nukem's CK Fixes first, 
  because there are actually a few pieces  of data that the public Creation Kit 
  forgets to convert properly, including some weapon data.

* If you've created  your content entirely  in SSE-supporting  tools, using SSE 
  data as a base, then you're probably fine. For example, if you use SSEEdit to 
  override data  in Bethesda's official ESMs that is version 43,  it's fine for 
  your override to be version 43 as well, so long as you're not adding new data 
  that takes advantage of SSE-only features  (e.g. if Bethesda were to add some 
  new subrecord, or some new format for an existing subrecord).

* If at any point you're worried that  you might cause issues for people if you 
  don't re-save your mod in the Skyrim  Special Creation Kit, then just re-save 
  it in the Creation Kit. Don't ship mods that you feel anxious about shipping.

### For users

* If you see a  version 43 override in a Skyrim Special Edition  mod, check the 
  version on the overridden  (master) record. If the  overridden record is from 
  official content  and it's also version  43, then it's  probably fine. If, on 
  the other  hand, a version  44 record is being  overridden with a  version 43 
  record, then  that's at  least a little weird, and  I wouldn't  fault you for 
  reaching out to the mod's author for clarification.

* If the mod in  question *isn't even a  port*, and if you  don't have specific 
  reason to believe  that something is wrong other than  the fact that a script 
  told you that the mod has *the bad number* in it, then don't panic.

* If you don't understand any of the  above, then you lack the expertise needed 
  to check whether a version 43 record is  likely to be harmful.

* Regardless of your expertise, if a version 43 record ever actually leaves you 
  feeling worried,  then just re-save the mod's ESP file in  the Skyrim Special 
  Creation Kit  (with CK fixes installed) yourself. Don't go  combing your load 
  order and bothering mod authors over it  unless you have a clear reason to be 
  concerned.