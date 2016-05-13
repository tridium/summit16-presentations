# Migrating from Niagara AX to Niagara 4
* Craig Gemmill & Blake Puhak - Software Engineers, Tridium, Inc.

Welcome to Developer Day.

Craig and Blake are Software Engineers at Tridium and have been working on the Niagara Framework for about 15 years now – starting with R2, moving on to AX and now Niagara 4.

This is the first of our presentations designed to get you up to speed with Niagara 4 development essentials.  We are going to show code and dig into development topics.  We hope you will be entertained a little, and learn enough to be able to develop confidently in the Niagara 4 environment.

We going to take you step-by-step through the process of converting a Niagara AX module to Niagara 4.

If you would like to see the code in more detail, you can check out our GitHub repo for all Niagara Summit code.  A link is included in the Summary section.

## Major Changes in Niagara 4

What’s new in Niagara 4 and why do we have to migrate our code?

Niagara 4 is an evolution of Niagara AX.  There were a number of changes that we needed to make to allow the Niagara Framework to integrate with modern technologies and set the foundation for the next 10 years of Niagara.

Between Java 4 and Java 8 a number of language features were added that we can now take advantage of.  adds: generics, annotations, try-with-resources, functions & lambda to name a few.  With the introduction of Java 8 we’re able to take advantage of compact profiles.  Even with all of the additional language level functionality, Java 8 compact 3 is about the same size as Java 5 SE and the compact 3 profile only eliminates APIs that we don’t typically use on a JACE such as Swing.  On the Supervisor full Java 8 SE is available.

In our efforts to embrace open technologies, we’ve adopted the gradle build tool.  Gradle is used by companies such as Netflix, LinkedIn and Google.  In integrates well into CI pipelines like Bamboo and Jenkins.  It offers great flexibility and allowed for the extensibility that we needed for the Niagara operations such as module-includes and custom permissions.

We took the opportunity in Niagara 4 to conform to industry best practices of separating application data from user data.  This means that the core runtime files of Niagara still live under C:\Niagara and this location is still referred to as Niagara Home, but the user configurable data like option files and stations now live under Niagara User Home.  When running Workbench, Niagara User Home is under your Windows (or Linux) User Home.  When a station is run via the Niagara Daemon, the Daemon’s User Home is under C:\Program Data (on Windows systems).

With some of our new security features in Niagara 4, namely code signing, we needed an alternative to Niagara AX’s module content filter.  We call this Runtime Profiles.  We’ll dig deeper into this is just a moment.

API Updates.   We changed a couple of our core APIs.  This allows us to use larger data sets more efficiently and replace some of our older APIs that are now provided by Java 8 like the Logger API.

## Niagara AX Module
Now that you’ve seen some of the changes we’ve made in Niagara 4, let’s begin our migration by taking a brief look at a simple Niagara AX module.  

Some of you may recall the Niagara Summit used to be held in Florida, and there you could occasionally catch a glimpse of alligators in the ponds, by the golf course, and if you were “lucky”, maybe on your patio!  Since the Niagara Summit has now come back to Gator Country, we have created a little Gator Tracker application in Niagara AX to watch for new alligators and record their length and weight.  

Our Gator Tracker includes:  

* A simple view to show the gators in a tabular form
* A history log of all gator's lengths with record gator lengths generating an alarm notification
* An action to write the list of gators to a file
* An action to execute a BQL query for display names

The code for this resides in three classes – BGator, which represents the gator object itself; BGatorTracker, which creates gators and generates alarm and history interactions; and BGatorTrackerView, which creates this Workbench view.  All of this is in the ax folder the AX to N4 Migration section of the GitHub repo.

So, what do we need to do to make our module compatible with Niagara 4?

## Source Code
### Directory Structure
We’ll get to the API changes later, but the first step is to get the module into the proper directory structure for a Niagara 4 module.  The most noticeable change is that modules in Niagara 4 are broken into different “module parts.”  The second change is that build.xml has been replaced by .gradle files.  Basically, we need to take the file tree on the left of the slide containing a single module and make it look like the one on the right containing two module parts.  

To do that we need to understand a little about module parts and runtime profiles.

## Runtime Profiles
Niagara 4 modules are made up of one or more module part jars.  Each of these module parts has a runtime profile that can be used to determine when to install the module part as part of a module install.

What is a Runtime Profile?  Niagara 4 introduced runtime profiles as a replacement for Niagara AX's module content filter.  We needed to do this for code signing since stripping class files our of a module at installation time would break the signature of the jar file.  We needed another solution so we introduced runtime profiles.  This does the same thing as the module content filter but at compile time rather than runtime and is a little more fine grained.

We’ve defined five runtime profiles.  Starting at the lowest profile, we have rt...

* rt contains the core data model and communications.  It's code that runs on a JACE and must be compiled against Java 8 compact 3
* ux contains Niagara 4 UI code, Javascript files, HTML, CSS and even Themes
* wb contains BajaUi views and Hx Views
* se contains any code that requires Java 8 SE to run
* doc contains any documentation

## Tips & Tricks
Now we know about the different runtime profiles that exist.  But before we start migrating our module, let’s go over a few rules and guidelines on using the different runtime profiles that will help us in our migration.

When you are migrating a Niagara AX module to Niagara 4, you will usually need to divide your packages into an rt and a wb module part (and possibly se and doc parts).  If you have a ux module part in your N4 module, that will be new code, as we did not have an equivalent in Niagara AX.

First, you cannot have a dependency on a module part that is “higher” in the stack, such as having an rt module with a dependency on a wb module.

Packages can exist only in a single runtime profile.  You cannot have the ui package in both rt and wb module parts.

When trying to decide if a particular class or package should go into your rt or your wb module part, you can use the Niagara AX build.xml.  If a package has the install=“ui” attribute on it in the build.xml, this indicates the package and its classes should go into the wb module part.

While lexicons can live in any or all of the module parts, we’ve found it’s easiest to just not split them up and put them in the rt module part.

One file that is a little special here is the type definitions file, the module-include.xml.  We’ll cover that in just a bit, after we get everything into our IDE.

## New Module Wizard
Let’s get started with our migration.  We can go ahead and create our new directory structure by hand and replace our build.xmls with new gradle files and copy over some boiler plate gradle files, but that’s a lot of work.  Let’s see how we can let Niagara tools do a lot of that work for us.

One thing we can do is use the New Module Wizard to create our new directory structures and gradle build files for us.  Since a typical Niagara module will be broken up into multiple module parts, we’ll need to run the New Module Wizard once for each module part.  Then we have some sample gradle code in the dev directory of the Niagara 4 Developer Image that will help us tie all of the module parts together so we can build them all at once.  That’s not too bad and gives us a big head start.

Recently we asked ourselves, “can we do better?”  We did some playing around and tried a couple of different things and we’d like to give you a little demo some changes that we’re working on that we think will make things easier.  This is a sneak peak and it’s not in any build yet, but we want to get your feedback on it to see if you think it will be helpful and if there are any other changes we can make so that developing in Niagara 4 is easier.

The big change that we’re going to give you a sneak peak of is having the New Module Wizard create multiple module parts at once as well as auto-generating more boilerplate gradle code needed for a multi-module build environment.

### Niagara AX build.xml
We’ll use our build.xml from the AX module to help determine which dependencies we need in our Niagara 4 module.  In order to select the correct module part(s) to depend on we need to think about how we're using our dependencies.  For alarm and history we're only using runtime components and not UI so we'll select only the rt runtime profile dependencies.

### New Directory Structure
Our New Module Wizard has now created two module parts with the Niagara 4 directory structure.  We can now copy our source files from our AX module to our Niagara 4 module.

* Copy source files into appropriate module part
* Copy lexicon/palette files into rt module part
* Don't copy module-include.xml (yet)

## Import into IDE
Now that our module source tree is all setup, let’s import it into our IDE so that we can make some edits.

Gradle offers some neat plugins that can help you get your development environment setup.  It can auto-create Eclipse project files for you, which you can then open in Eclipse and already have your project set up.  To do this open up a Niagara console, navigate to your module development environment and run “gradlew eclipse.”  It will auto-create you Eclipse project files for you.

At Tridium, most of our developers switched over to using IntelliJ IDEA when they made the switch to Niagara 4.  Even those (like Craig) who were strong Eclipse supporters have found that IntelliJ IDEA is easier to use, so we’re going to use that for the remainder of this presentation.  The transition to IntelliJ is really not difficult at all – in fact, you can even tell IntelliJ to use the same hot-keys that you’re used to using in Eclipse.

Like with Eclipse, we can run “gradlew idea” and have the gradle plugin auto-create our IntelliJ IDEA project files for us. But when using IDEA 2016.1 with these New Module Wizard changes, it’s even simpler.  Simply open IntelliJ and point it at your project’s root build.gradle file.  No extra commands needed.

## Gradle Build Commands
### Using the Gradle Wrapper
We’ve mentioned gradle several times now and many of you may have tried using it already.  Despite the angry elephant logo, gradle is not all that scary.

For most of your day-to-day work, you only need a few simple commands (that look very similar to their AX equivalents).  

The Building with Niagara 4 presentation goes into more details about gradle and all of the cool things you can do with it, but for now we’ll keep things simple and use gradlew jar to compile our module.  In fact, we’ll just use this from within the IDE, so it’s even easier!

## Fixing module-include
### Option 1

Now you may recall earlier, we said we were going to hold off on doing anything with the module-include file.  Well now it’s time - we need to get the type definitions assigned into the correct module parts - a type can only be defined in a single module part.  

We can do this in two ways.  For a little project like this, with only a few files, it doesn’t matter, but for a large module, it will definitely make a difference.  

The first way is to hand-edit the module-include.xml in each module part to include only the types that are included in that module part.  But this is risky as it’s very easy to make a mistake in typing the XML by hand.  And your mistake may not be caught until runtime deployment when the types you messed up are stripped from your users' station databases!

### Option 2: Let the Tools Work for YOU!
A better solution is to let the tools do it for us!

In Niagara 4.2, we’re introducing an enhancement to the slot tool to convert your AX slot-o-matic code to Niagara 4 slot annotations.  This tool will also help us out with our module-include type definition problem!

First, we’ll copy the module-include.xml from the AX location, into BOTH the rt and the wb module parts.  We do this because the type definitions in module-include are used by the slot tool in migration to figure out what annotations to insert in each file (most notable agent-on annotations).

Next, we run the handy slot-o-matic migration task defined for us in our gradle window.  You can also run this from the command line if you like.  Looking at our BGator class, you can see that the old syntax has been replaced with shiny new annotations!  Even better, look at the colors – that means the IDE understands the syntax!  And it can now help me out with all the fancy comforts of modern editors like auto-completion and syntax correction.

Now, we can remove the module-include files that we copied over, because they are no longer needed – the build tool will regenerate them with the correct information for us.  Remember, a type can only be defined in a single module part, so we need to remove the duplicates that are defined in the identical module-include files.

And now we can build our module!  Let’s try it now.

## Slot-o-matic
### Now with Gradle
Let’s take a brief look at the difference in syntax between the Niagara AX slot code, and the Niagara 4 slot annotations.

In Niagara AX we used a proprietary format, with a unique comment syntax, to identify the slot-o-matic code.  This could not be recognized by any IDEs, so you had to type this all on your own.  In Niagara 4 we use straight Java annotations.  We’ve written code to define the behavior of these annotations, but because it is a Java-understood construct, the IDE can help us out.

The Gradle slot-o-matic tool still recognizes the AX-style slot-o-matic syntax, and will parse it and create Properties, Actions and Topics.  But if you want to be one of the cool kids, you’ll convert your slot-o-matic code to use Annotations.  Inside of an IDE, they are much easier to read and enter – IntelliJ will offer auto-completion suggestions to speed up your typing.  Another bonus is that when using annotations, the build process will now automatically create your module-include.xml file for you!

## Build Errors
### Collections API
The first thing we notice is that the build failed.  Why can’t we use BICollection?

BICollection no longer exists.  This is an older API that made it really difficult to create new collections.  The main problem with BICollection is that every implementation also needs to implement BIList and BITable.  So we simplified things and just kept BITable, getting rid of BICollection & BIList.  We've also eliminated the random access methods to BITable and only Cursor based access remains.  This encourages us to parse through our data sets in a more efficient manner.  A handy side effect of BICollection's removal is that BQL queries now resolve directly to a BITable.

### Alarm & History APIs
The collections problem is now fixed.  Now, it looks like something must have changed in Tridium’s alarm database too.

We made some big changes to the Alarm and History APIs.  When connecting to remote databases, the old APIs were very inefficient in their connection handling and would open and close connections for every API call.  To fix this for Niagara 4, we introduced HistoryDb and AlarmDbConnections for interactions with the databases.  So instead of calling getRecord() on the database, you now get a connection to the database, then call getRecord() on the connection.

We also need to be careful about closing our connections.  Since we can now use the Java 7 APIs, I am going to follow the best practice of wrapping my connections in a try-with-resources statement.

While we’re in here, the code is using javax.baja.Log, which should be updated.  In Niagara 4, logging is done through the standard java.util.logging.Logger class.  Existing code will work as is, through a wrapper class, but again, best practice is to update it.  I won’t do that all here, but you can see the result in the clean version in the GitHub repo.

### Incorrect Runtime Profile Import
It looks like we can’t find BInsets.  Tridium didn’t move or remove that class in Niagara 4, so perhaps we selected the wrong dependency.  Let’s look at the dependencies for out nola-wb module part.  We selected gx-wb, but I think we really wanted gx-rt.  A lot of the basic gx code like BInsets and BColor are in the rt runtime profile because they don't have any dependencies on other UI code that's in wb module parts.

## Station Migration
Before we can test our migrated module in Niagara 4 we need to migrate the station we were using in AX.  Station Migration involves the conversion of AX artifacts to the corresponding N4 format; for example, station .bog files, Px files, and the history and alarm databases.  The bog migration will update the slots on the Web Service, add the new Role Service and handle any updates needed specified by the bog element converters for any changed types.  The migration tool will also make any necessary changes to references in your Px pages so they will work in Niagara 4.  One thing it won’t do is migrate the Px pages into bajaux for you; that you’ll have to do yourself.

The Station Migration process is covered the Applications Track talk with John Brown and Keith Goodson (as well as in the Niagara 4 documentation), so we won’t go into too much detail here.  The executable is n4mig, and if you run it with no arguments at the console command prompt of the Niagara 4 installation you are using, it will display the options and parameters available and required.  For those of you not familiar with the tool, it takes a Niagara AX station, generally in the form of a Niagara backup distribution file, and converts the station inside along with all of its descendant elements to a Niagara 4 version.

The reason for using a dist file instead of just a bog file, is so that the exact version of each module used in the station can be known.  This allows the migration tool to be able to apply exactly the correct conversions for that version of the Niagara module that defines each type.  Without that information, which is contained in the distribution manifest, the tool would have to make a guess at the version, and that could easily spell trouble.

## Making Your Own Niagara 4 Changes
While you're porting your module from Niagara AX to Niagara 4, you may find that now is a good time to make some changes in your module that you could not make from one AX version to the next.  In general you want to avoid making breaking changes like changing slot names and types, or changing Niagara Type names, because this makes it difficult for a station using the old version of your module to be run using the new version.  However, in Niagara 4 the migration tool was designed with an extensible mechanism in mind, so that you can make these changes to your Niagara Types, and implement one of our pluggable migration classes to handle the transition for user stations.

When you change a Niagara Type that appears in bog files, you will need to implement a BIBogElementConverter to handle the conversion to user stations.  When you change a the format of a file type that your module uses, you will need to implement a BIFileMigrator to handle that conversion.  

## Summary
While we needed to make a number of changes to convert our Niagara AX module to Niagara 4, none of the changes we made were all that difficult.  We hope you learned something during this presentation that this gives you a little more confidence in being able to convert your Niagara AX modules to Niagara 4.

Just a reminder, all of the code in this presentation along with the presentation slides are located in our github repo at [https://github.com/tridium](https://github.com/tridium)
