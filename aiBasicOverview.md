# RimWorld AI introduction
As it is with many things, determining what one *can* do is easy, but determining what one *should* do is hard. RimWorld is no exception. This tutorial hopes to explain how RW can determine what actions (Jobs) a Pawn can perform, and by what means (ThinkTree) that Pawn determines which of those actions to do at any given time. Furthermore some basic examples of how to extend/implement certain behavior will be discussed.

## Pawn Actions are Jobs
In RW whenever a Pawn, either player controlled or AI controlled, performs an action (sleeping, eating, going to drafted location, work, etc) they are in fact performing a **Job**. Jobs are the fundamental element of action for Pawns in RW, and thus the immediate responsiblity for the AI system is to assign jobs to each Pawn.

## Whether a Pawn can perform a job
The actual logic in how to assign a job/whether that job is assignment are contained within the various *Giver* classes: subclasses of ThinkNode_JobGiver and WorkGiver. Below is a very basic JobGiver

```csharp
public class JobGiver_GetJoyInBed : JobGiver_GetJoy
{
	...

	protected override Job TryGiveJob(Pawn pawn)
	{
		if (pawn.CurJob == null || pawn.CurrentBed() == null || !pawn.Awake() || pawn.needs.joy == null)
		{
			return null;
		}
		float curLevel = pawn.needs.joy.CurLevel;
		if (curLevel > 0.5f)
		{
			return null;
		}
		return base.TryGiveJob(pawn);
	}
}
```
As one can see, a Pawn cannot start GetJoyInBed unless they are currently doing something, are in bed, are awake, and need Joy. Without diving into too much detail (Check out [Mehni's Job Tutorial](https://github.com/Mehni/ExampleJob/wiki)), it should be noted that the a Pawn cannot do a Job *iff* that Job's Giver returns an invalid job. If a valid job is returned, that Job **will** be assigned. 

## Work vs Jobs
Work is considered to be a special type of Job, subject to work type categorization (via the work tab) as well as pawn restrictions. All Work is provided by subclasses of `WorkGiver`, broken up into two large categories: `WorkGiver_Scanner` and everything else. Types of work where the actual target of that work can vary (which plant to harvest, which wall to build, etc) should be implemented as a subclass of `WorkGiver_Scanner` (as that WorkGiver will scan/look for possible work). The relevant parts of a simple `WorkGiver_Scanner` are listed below

```csharp
public abstract class WorkGiver_Haul : WorkGiver_Scanner
{
	public override Danger MaxPathDanger(Pawn pawn)
	{
		return Danger.Deadly;
	}

	public override Job JobOnThing(Pawn pawn, Thing t, bool forced = false)
	{
		if (!HaulAIUtility.PawnCanAutomaticallyHaul(pawn, t, forced))
		{
			return null;
		}
		return HaulAIUtility.HaulToStorageJob(pawn, t);
	}
}
```
Similarly to the JobGivers, if JobOnThing returns a valid job then that job is taken. Two differences with the standard JobGivers is the lack of the `TryGiveJob` idiom (WorkGiver_Scanners have a method `HasJobOnThing` that default to seeing if `JobOnThing() == null`) and inclusion of the `forced` parameter, which corresponds to whether the player directly forced that action via right click. There are several other methods useful to a WorkGiver_Scanner, and a similar approach to the above is used to target cells and not things.

*NOTE* If you want to only allow actions taken directly by the player, require `forced = true` in the WorkGiver.

## Determining Pawn behavior
In short, a Pawn will evaluate a set of JobGivers/WorkGivers and will execute the first valid job that is returned. Thus, the entirety of RW pawn behavior can be reduced to 2 characteristics: whether any given JobGiver/WorkGiver is valid for that pawn, and the order in which these Givers are checked. The former has been discussed, and the latter is contained within a Pawn's ThinkTree.

*NOTE* Its worth considering that Area restrictions, time of day restrictions, and work priorities interact only with the various Givers and not with the ThinkTree itself.

## ThinkTree basics
The various Givers for a pawn are configured in a general tree structure, allowing individual nodes (`ThinkNodes`) to have up to 1 parent, and an arbitrary number of siblings/children. Each tree will have a root Node that begins evaluation, and there are several flow control nodes such as `ThinkNode_Priority` (in order), `ThinkNode_PrioritySorter` (sorted by child priority) or `ThinkNode_Random` (random) that determine the execution order of children nodes. ThinkTrees can also be inserted as `ThinkNode_SubTrees` into other ThinkTrees as well. The simplest RW full ThinkTree (Mechanoid.xml) is show below, note the inclusion of the SubTrees `Downed` and `LordDuty`

```xml
<?xml version="1.0" encoding="utf-8" ?>


<Defs>

  <ThinkTreeDef>
    <defName>Mechanoid</defName>
    <thinkRoot Class="ThinkNode_Priority">
      <subNodes>
        <!-- Downed -->
        <li Class="ThinkNode_Subtree">
          <treeDef>Downed</treeDef>
        </li>
        
        <!-- Do a queued job -->
        <li Class="ThinkNode_QueuedJob" />

        <!-- Lord -->
        <li Class="ThinkNode_Subtree">
          <treeDef>LordDuty</treeDef>
        </li>

        <!-- Idle -->
        <li Class="ThinkNode_Tagger">
          <tagToGive>Idle</tagToGive>
          <subNodes>
            <li Class="JobGiver_WanderAnywhere">
              <maxDanger>Deadly</maxDanger>
            </li>
          </subNodes>
        </li>
        
        <!-- Idle error -->
        <li Class="JobGiver_IdleError"/>
      </subNodes>
    </thinkRoot>
  </ThinkTreeDef>
  
</Defs>
```

This ThinkTree says to, in order, perform any valid actions from the `Downed` think tree, then any `LordDuty`, then to execute `JobGiver_WanderAnywhere` with the "Idle" tag. Finally there is an error catch, suggesting that `JobGiver_WanderAnywhere` should always return a valid job.

Just as Jobs are the fundamental element of Pawn action, ThinkTrees are the fundamental element of pawn behavior in RW. To influence pawn behavior beyond the conditions whether or not a Job can be taken (or to introduce new work) will require the alternation/creation of some ThinkTree.

ThinkTrees themselves are particular instances of ThinkTreeDefs that are replicated and assigned upon pawn creation, and are configured within a Pawn's RaceProps. Once a pawn has been created, their ThinkTree will remain static and unchanged; the only changing part belonging to a pawn's assigned Lord DutyDef (more on that later).

## Constant vs non-constant ThinkTrees

Each pawn will actually possess 2 ThinkTrees, their primary ThinkTree and a ConstantThinkTree. Primarily this constant ThinkTree is a reduced set of autonomous responses that will be evaluated first. As an example 

```xml
<ThinkTreeDef>
  <defName>HumanlikeConstant</defName>
  <thinkRoot Class="ThinkNode_Priority">
    <subNodes>
      <li Class="ThinkNode_ConditionalCanDoConstantThinkTreeJobNow">
        <subNodes>
          <!-- Flee explosion -->
          <li Class="JobGiver_FleePotentialExplosion" />
      
          <!-- Hostility response -->
          <li Class="JobGiver_ConfigurableHostilityResponse" />
          
          <!-- Lord directives -->
          <li Class="ThinkNode_Subtree">
            <treeDef>LordDutyConstant</treeDef>
          </li>
        </subNodes>
      </li>
    </subNodes>
  </thinkRoot>
</ThinkTreeDef>
```

Thus the most pressing concerns for a Pawn are fleeing an explosion and their basic hostility response. 

*NOTE* When drafted, a pawn will not evaluate their constant ThinkTree, as their autonomy has been removed.

*NOTE* Any logic placed in a ConstantThinkTree should be easily interruptible, preferably stateless

## How To Manipulate A ThinkTree
As stated before, a complete description of a Pawns behavior can be broken down into 2 dimensions: the first is whether any particular task/job is available to them (the Giver returns a valid job), the second is what order these Tasks/Jobs are attempted (ThinkTree structure). So the question can be asked, what freedom do we possess to alter the ThinkTree?

First and foremost, ThinkTrees are simply xml ThinkTreeDefs and can be handled as any other defs. Creating a new ThinkTree and/or patching an existing one are both very useful strategies; patching in particular can be very granular but also rather tricky. Otherwise, there exist certain tagged hooks in the standard Human and Animal ThinkTrees that will insert a subTree into one of various places as determined by the specific tag. To utilize this functionality, create a ThinkTreeDef with `<insertTag>TAG</insertTag>` with the following possible values of TAG
* Humanlike_PostMentalState
* Humanlike_PostDuty
* Humanlike_PreMain
* Humanlike_PostMain
* Animal_PreMain
* Animal_PreWander

Any ThinkTree such tagged will be inserted into the main Animal/Humanlike ThinkTrees, as well any ThinkTrees that use `ThinkNode_SubtreesByTag` and the appropriate tag.

Presently if you wish to add behavior to a particular PawnKindDef or new race, there are 2 primary options. First is to create a custom ThinkTreeDef and include it in the race properties; this is a good option if the behavior is radically different (simple is welcome too). Secondly one can create a ThinkTreeDef to insert with an `<insertTag>`, with that ThinkTreeDef either using custom `ThinkNode_Conditional`s or Givers that will return an invalid job when attempted with a Pawn that is not of that Kind/Race.

*NOTE* If taking the latter course of action, place those checks immediately in the Giver, so that your ThinkTree does not needless slow down unintended pawns.

## The final piece for pawn behavior, Duties
Previously it was mentioned that a pawn's ThinkTree is mostly static (the previous section describes changes that are always present), saving for that pawn's duty. But what is a duty? It is a DutyDef, but from the pawn's perspective it is a ThinkTree and a cell (called the focus). This is how to have Pawns perform higher, more organized actions than a simple Job; especially when combined with the focus or via repetition. An example duty is listed below, that for AI pawn following a character
```xml
<DutyDef>
  <defName>Follow</defName>
  <thinkNode Class="ThinkNode_Priority">
    <subNodes>
      <li Class="JobGiver_AIFollowEscortee" />
      <li Class="ThinkNode_ForbidOutsideFlagRadius">
        <maxDistToSquadFlag>16</maxDistToSquadFlag>
        <subNodes>
          <li Class="ThinkNode_Subtree">
            <treeDef>SatisfyBasicNeedsAndWork</treeDef>
          </li>
        </subNodes>
      </li>
      <li Class="JobGiver_WanderNearDutyLocation">
        <wanderRadius>5</wanderRadius>
      </li>
    </subNodes>
  </thinkNode>
</DutyDef>
```
From above, this has a pawn follow the escortee, it then attempts to `SatisfyBasicNeedsAndWork` by loading the appropriate subTree with a distance restriction (how this works specifically is left to the reader), finally wandering in the vicinity of the duty location (focus). There is similarly a Duty for each of the various AI attacker styles, prisoner breaks, essentially any concerted set of tasks that a pawn will perform autonomously in response to some event / higher logic / raison d'être (sappers gonna sap). As to where Duties come from, well ... until next time!
