# RTS PROTOTYPE
Prototype for a survival strategy game where light is required to dispel the darkness

[![Watch the video](https://i9.ytimg.com/vi_webp/vfWqTihb8FY/mq3.webp?sqp=CJDPm9QG-oaymwEmCMACELQB8quKqQMa8AEB-AH-CYAC0AWKAgwIABABGGUgXyhKMA8=&rs=AOn4CLBY-ycoaL38ImlF5_SbWTo696RXIg)](https://www.youtube.com/watch?v=vfWqTihb8FY)

Uses UI Binding package for all UI:
[UI Binding Github](https://github.com/paulkelly/UIBinding)

## Index
[Procedural Generation](#Procedural-Generation)

[Unit Commands](#Unit-Commands)

[Grid Shader](#Grid-Shader)

[Fog Of War](#Fog-of-War-Calculations)

[Unit Attack Target Calculations](#Unit-Attack-Target-Calculations)

## Procedural Generation

The terrain is generated using perlin noise to generate areas of forest, water and resources then poisson disc sampling to place objects like trees and rocks.

## Unit Commands

In most RTS games, clicking on a unit will display commands unique to that unit.

I wanted to create a system that would allow me to define commands in a way that would be easily configurable.

<img width="266" height="184" alt="20240819212005" src="https://github.com/user-attachments/assets/97426917-66b6-4cbf-8f77-b87318354472" />

### The Idea

A Command scriptable object is used to manage the visuals for each Command, as well as hold an event that units can subscribe to if they are ready to respond to that command.

This command object can then be bound to the UI.

Another scriptable object, the Command Template is used to define what Commands a unit has and the order those commands should appear in the UI.

A unit has a reference to a Command Template, along with scripts that subscribe to each Command and implement some code to execute when that Command is pressed.

<img width="730" height="455" alt="20240820003741" src="https://github.com/user-attachments/assets/e3f8dcef-f2a8-4d05-b5da-4eefa12ce18d" />

### How Adding a New Command Works
**Create a new command object**

Use the Create menu to create a new scriptable object and assign images for the button.

<img width="722" height="281" alt="20240819213747" src="https://github.com/user-attachments/assets/947bea63-b638-48a9-b03d-a9653d744b90" />

**Add to a Template**

<img width="725" height="551" alt="20240819213945" src="https://github.com/user-attachments/assets/bc92a91c-9d47-4ff3-aef6-0820e50039ab" />

### Write Code for Units to Implement the Command

```c#
public class DebugCommandListener : MonoBehaviour, ICommandRegister 
{  
    [SerializeField] Command command;  
    
    public void Register()  
    {        
	    command.Register(RespondToDebugCommand);  
    }  
    public void Deregister()  
    {        
	    command.Deregister(RespondToDebugCommand);  
    }
      
    public void RespondToDebugCommand()  
    {        
	    Debug.Log("Unit has received debug command");  
    }
}
```
Attach script to our unit and configure the command scriptable object. This could also be configured globally (e.g. In a static singleton with reference to all commands).

<img width="455" height="74" alt="20240819214418" src="https://github.com/user-attachments/assets/3927b686-1a80-465f-bf4b-3173268bf678" />

### Finished Adding a New Command

This command could now be assigned to multiple units, and easily moved to a new position.

<img width="1199" height="527" alt="20240819235121" src="https://github.com/user-attachments/assets/2dfd1017-5178-4ce4-934d-b4e22ce870a8" /><img width="716" height="204" alt="20240819235859" src="https://github.com/user-attachments/assets/7a5c4e9f-284f-41da-8a89-6ba89cd48f2f" />


### The Command Scriptable Object

First there is the `BaseCommand` script that defines the images and an Execute Method. The Execute Method will be called by the UI when the button is clicked (or a hotkey pressed).

```c#
[Serializable]  
public abstract class BaseCommand : ScriptableObject  
{  
    public Sprite image;  
    public Sprite imageHover;  
    public abstract void Execute();  
}
```

Next there are two implementations, a regular command for simple actions (such as the Debug Command created here) and a generic command that can be used to create more complex commands requiring data. The Move command for example is a `Command<Vector3>`.

These scripts contain an event that can be subscribed to. When a unit is selected it subscribes to all commands it has been configured for. This allows all selected units to respond to a command.

The simple command immediately calls the event when pressed.

```c#
[CreateAssetMenu(menuName="Command/Command")]  
public class Command : BaseCommand  
{  
    private event ExecuteCommand OnExecute = () => { };  
  
    public delegate void ExecuteCommand();  
  
    public override void Execute()  
    {        
	    OnExecute.Invoke();  
    }        
    public void Register(ExecuteCommand action) => OnExecute += action;  
    public void Deregister(ExecuteCommand action) => OnExecute -= action;  
}  
  
public abstract class Command<T> : BaseCommand  
{  
    private event ExecuteCommand OnExecute = (value) => { };  
  
    public delegate void ExecuteCommand(T value);  

	// Does not implment Execute()
	// Any implementation still needs to implement this method
  
    public void ExecuteWithValue(T value)  
    {        
	    OnExecute.Invoke(value);  
    }        
    public void Register(ExecuteCommand action) => OnExecute += action;  
    public void Deregister(ExecuteCommand action) => OnExecute -= action;  
}
```


### Position Commands

The Move command is an instance of this Position Command, which requires a Vector3.

```c#
[CreateAssetMenu(menuName = "Command/Position Command")]  
public class Vector3Command : Command<Vector3>  
{  
    public override void Execute()  
    {        
	    SceneReferences.Instance.inputHandler.SetCommand(this);  
    }
}
```

Rather than immediately invoke the `OnExecute` event, it first makes a call to the Input Handler.
When the command is executed, the next click will order the unit to move to that position.

```c#
public void SetCommand(Vector3Command command)  
{  
    HasTargetCommand = true;  
    currentTargetCommand = command;  
}

...

// Called in Update if HasTargetCommand and mouse pressed this frame
private void HandlePositionCommand(Ray ray) 
{      
    int hits = Physics.RaycastNonAlloc(ray.origin, ray.direction, _hits, MaxDistance, groundLayers);  
  
    if (hits > 0)  
    {
	    currentTargetCommand.ExecuteWithValue(_hits[0].point);  
    }    
    
    HasTargetCommand = false;  
    currentTargetCommand = null;  
}

```


### Command Templates

This is attached to a script on each unit, and determines which commands should be shown when they are selected

```c#
[CreateAssetMenu(menuName = "Command/Command Template")]  
public class CommandTemplate : ScriptableObject 
{   
    public BaseCommand[] commandRow1 = new BaseCommand[5];  
    public BaseCommand[] commandRow2 = new BaseCommand[5];  
    public BaseCommand[] commandRow3 = new BaseCommand[5];  
}
```

### The Units

<img width="716" height="204" alt="20240819235859" src="https://github.com/user-attachments/assets/6ad9fa6c-0465-4ec3-917e-e6fd4669614c" />

The units are configured with a number of scripts with the `ICommandRegister` interface. `MoveableEntity` implements the Move and Stop commands.

```c#
public interface ICommandRegister  
{  
    public void Register();  
    public void Deregister();  
}
```

The Selectable Object script gathers a reference to all scripts with this interface and when selected, they register to all commands.

```c#
[SerializeField] private CommandTemplate commands;

private ICommandRegister[] commandListeners;  
  
private void Start()  
{  
    commandListeners = GetComponents<ICommandRegister>();  
}  
  
private void RegisterCommands()  
{  
    SceneReferences.Instance.commandButtonGrid.Bind(commands);  
    foreach (var commandListener in commandListeners)  
    {        
		commandListener.Register();  
    }
}  
  
private void DeregisterCommands()  
{  
    foreach (var commandListener in commandListeners)  
    {        
	    commandListener.Deregister();  
    }
}

public void OnSelect()  
{  
    ...

	// This is a temporary measure to tell the UI to use this units commands
	// When multiple units can be selected at once this will need a system to
	// handle combining templates from all selected units
	SceneReferences.Instance.commandButtonGrid.Bind(commands);
	
    RegisterCommands();  
}  
  
public void OnDeselect()  
{  
    ...
    DeregisterCommands();  
}
```

### UI

Finally the UI has a list of button objects and iterates through each of them, binding them to the corresponding Command.

```c#
public void Bind(CommandTemplate unitCommands)  
{  
    if (unitCommands == null)  
    {        
	    Unbind();  
        return;  
    }    
    int next = 0;  
    for (int i = 0; i < unitCommands.commandRow1.Length; i++)  
    {        
	    commandBinders[next].Bind(unitCommands.commandRow1[i]);  
        next++;    
    }    
    for (int i = 0; i < unitCommands.commandRow2.Length; i++)  
    {        
	    commandBinders[next].Bind(unitCommands.commandRow2[i]);  
        next++;    
    }    
    for (int i = 0; i < unitCommands.commandRow3.Length; i++)  
    {        
	    commandBinders[next].Bind(unitCommands.commandRow3[i]);  
        next++;    
    }
}
```

```c#
public class CommandBinder : MonoBehaviour  
{  
    [SerializeField] private Button button;  
    [SerializeField] private Image icon;  
    [SerializeField] private HighlightOnHover hover; //switches icon when hovered 
    
    public void Bind(BaseCommand command)  
    {        
	    Unbind();  
        if (command == null) return;
           
        icon.enabled = true;  
        icon.sprite = command.image;
        
        button.enabled = true;  
        button.onClick.AddListener(command.Execute); 
        
        hover.SetSprites(command.image, command.imageHover);  
    }  
    
    protected void Unbind()  
    {        
	    icon.enabled = false;  
	    button.enabled = false;  
        button.onClick.RemoveAllListeners();  
    }
}
```


## Grid Shader

## Fog of War Calculations

To get our Fog of War working we need to keep track of what areas of the map the player is able to see. Once we know what grid cells are visible, we can write that data to a texture and use a shader to render the fog of war.

### Resources:

[implementing fog of war for rts game in unity](https://blog.gemserk.com/2018/08/27/implementing-fog-of-war-for-rts-games-in-unity-1-2/)

[riot games fog of war story](https://technology.riotgames.com/news/story-fog-and-war)

[visibility](https://www.redblobgames.com/articles/visibility/)

### Basic Solution

To do this we can keep count of how many units are able to see each cell.

The most simple solution is to iterate over every unit, remove them from each cell they can see, update their position and re-add them with the new position.

To find which cells a unit can see we can loop over a grid from -radius to +radius, centred around the unit. Then test if each cell overlaps with the vision circle.

'unitsWithVisionInCell' is a flattened 2d array where [x,y] = [y*width + x]


``` c#
public void Update()
{
	foreach(unit in allUnits)
	{
		UpdateUnitVisibility(unit.gridPos, unit.radius, false);
		unit.gridPos = WorldToGridPos(unit.position);
		UpdateUnitVisibility(unit.gridPos, unit.radius, true);
	}

	UpdateTexture();
}

private void UpdateUnitVisibility(GridCell pos, int radius, bool add)
{
	for(int x = pos.x-radius, x < pos.x+radius; x++)
	{
		for(int y = pos.y-radius, y < pos.y+radius; y++)
		{
			GridCell cell = new GridCell(x, y);
			if(!cell.Overlaps(pos, radius)) continue;
			
			if(add)
			{
				unitsWithVisionInCell[y * GridBounds + x]++;
			}
			else
			{
				unitsWithVisionInCell[y * GridBounds + x]--;
			}
		}
	}
}
```


### Optimisations

**Only Update Units that Move**

Easy enough, if a unit doesn't move skip recalculating it's visibility 

```c#
public void Update()
{
	foreach(unit in allUnits)
	{
		GridPos newPos = WorldToGridPos(unit.position);
		if(unit.gridPos.Equals(newPos)) continue;
		
		UpdateUnitVisibility(unit.gridPos, unit.radius, false);
		unit.gridPos = newPos;
		UpdateUnitVisibility(unit.gridPos, unit.radius, true);
	}

	UpdateTexture();
}
```

**Make the Grid Smaller**

For a unit with a range of 20 units, if the Grid Size is 1x1 then to calculate it's vision requires 20x20 = 400 comparisons. If we change the Grid Size to 2x2, then we only need 10x10 = 100 comparisons.

So we perform 4x better if we reduce the Grid Size by 2x.

This is obviously a balance between gameplay and performance as having too big a grid size will make the fog of war blocky, and potentially affect other systems e.g. if the Building System uses the same grid.

**Optimising the Calculation**

Rather than iterate over a radius x radius grid and calculate if each cell overlaps, we can do better. Instead we calculate the height of each column, and only iterate over cells that are inside the circle.

[Bresenham's Algorithm](http://members.chello.at/~easyfilter/bresenham.html)

[Wikipedia](https://en.wikipedia.org/wiki/Midpoint_circle_algorithm)

```c#
private void UpdateUnitVisibility(GridCell pos, int radius, bool increment)  
{  
	int radiusSquared = radius * radius;
    for (int x = pos.x - radius; x < pos.x + radius; x++)  
    {  
        int xOffset = x - pos.x;  
        int height = (int)Math.Sqrt(radiusSquared - (xOffset * xOffset));  
  
        for (int y = pos.y - height; y < pos.y + height; y++)  
        {
  
            if (increment)  
            {
	            unitsWithVisionInCell[y * GridBounds + x]++;  
            }            
            else  
            {  
                unitsWithVisionInCell[y * GridBounds + x]--;     
            }  
        }
    }
}
```

### Further Improvement to the Calculation

If we know a unit has moved 1 grid space to the right, we know that most of the cells they occupy will remain the same. If we can calculate only the cells that need to change, we can reduce the number of cells that need to be considered.

<img width="488" height="480" alt="Pasted image 20240901162525" src="https://github.com/user-attachments/assets/6cfa37f5-8066-4a42-b794-ec826402f208" />

I have left this optimisation for now, as it makes the code quite a bit more complicated, and after moving things to the Job system, more optimisation wasn't required.

### Unity Job System

[Unity Job System](https://docs.unity3d.com/Manual/JobSystem.html)

Using the Job System will allow us to move our calculations off the main thread, essentially making it free so long as it finishes before we need to use the result.

The job system works best when used with the burst compiler, which does not allow managed types.

### The Job class

To work with the burst compiler our data needs to be converted to a structs and NativeArrays. 

The job will take in an array of Units that need to recalculate their vision, with the current and previous grid cell positions. A count is given to determine how far into the list to iterate.

The job outputs a flattened 2d array with the number of units in a given cell.

If possible we would like our jobs to be run in parallel, using IJobParallel rather than IJob. In this case, because each execution of the job wants to write to the output array in an undetermined order (i.e. units[0] does not exclusively write to unitsWithVisionInCell[0]), we cannot ensure thread safety and have to use a single job.

```c#
public struct UnitVision  
{  
    public GridCell lastGridCell;  
    public GridCell newGridCell;  
    public int radius;  
}
```

The method 'UpdateUnitVisibility' is the same as before.

``` c#
public struct UpdateUnitVisibilityJob : IJob  
{  
    [ReadOnly] public NativeArray<UnitVision> units;  
    public NativeArray<uint> unitsWithVisionInCell;  
  
    public int UnitCount;  
    public int GridBounds;

	public void Execute()  
	{
		for (int i = 0; i < UnitCount; i++)  
		{  
		    UnitVision unit = units[i];
			int radiusSquared = radius * radius;
			UpdateUnitVisibility(unit.lastGridCell radius, radiusSquared, false);  
			UpdateUnitVisibility(unit.newGridCell, radius, radiusSquared, true);
		}
	}

	private void UpdateUnitVisibility(GridCell pos, int radius, int radiusSquared bool increment)  
	{
		//...
	}
}
```

### The Schedular

To run the job, we need to create an instance of it and run '.Schedule()'. This returns a job handle, and at some point we need to call .'Complete()' on the handle to ensure the job is finished.

Our 'GridVisibilityManager' starts a job early in the frame, and then uses LateUpdate() to complete the job.

The data being used by the job, in this case the 'unitsToProcess' and 'unitsWithVisionInCell' cannot be used while the job is running. Attempting to do so will cause the job to be finished on the main thread. This will need to be considered when using jobs. 

One strategy if the data needs to be freely available would be to create a copy that can be accessed at any time, and when the job finishes the data can be synced.

In our case, we intend to use the data in another job, if this is the case we can use the previous job handle when scheduling the next job. This tells Unity that the next job depends on the previous one and the previous job must be completed first.


```c#
public class GridVisibilityManager : MonoBehaviour
{
	public const int WorldSize = 1024;  
	public const int GridSize = 4;
	
	public const int GridBounds = WorldSize / GridSize;

	public const int MaxUnits = 100;

	private NativeArray<UnitVision> unitsToProcess;  
	private NativeArray<uint> unitsWithVisionInCell;

	private readonly HashSet<Unit> allUnits = new HashSet<Unit>();

	private JobHandle _updateLitCellsJobHandle;  
	private bool _jobRunning;

	public void RegisterUnit(Unit unit)  
	{  
		// when units are added, the job should skip the decrement step
		// left out for brevity
	    allUnits.Add(unit);  
	}  
	  
	public void DeregisterUnit(Unit unit)  
	{  
		// when units are removed, the job should skip the increment step
		// left out for brevity
	    allUnits.Remove(unit);  
	}

	private void Start()  
	{  
	    unitsToProcess = new NativeArray<UnitVision>(MaxUnits, Allocator.Persistent);  
	    unitsWithVisionInCell = new NativeArray<uint>(GridBounds * GridBounds, Allocator.Persistent);  
	}
	  
	private void OnDestroy()  
	{  
	    if (!_updateLitCellsJobHandle.IsCompleted)  
	    {        
		    _updateLitCellsJobHandle.Complete();  
	    }    
	    
	    unitsToProcess.Dispose();  
	    unitsWithVisionInCell.Dispose();  
	}


	private void Update()  
	{  
	    int processQueueIndex = 0;  
	    foreach (var unit in allUnits)  
	    {
	        // skip if unit hasn't moved  
	        GridCell newPos = GridCell.FromWorldPos(unit.Position);  
	        if(unit.GridPosition.Equals(newPos)) continue; 
	         
	        unitsToProcess[processQueueIndex] = new UnitVision()  
	        {  
	            newGridCell = newPos,  
	            lastGridCell = obj.GridPosition,  
	            radius = obj.Radius
	        };  
	  
	        obj.GridPosition = newPos;  
	        processQueueIndex++;
	        
			if(processQueueIndex >= MaxUnits) break;  
	    }  
	    
	    if (processQueueIndex > 0)  
	    {        
		    var job = new UpdateUnitVisibilityJob()  
	        {  
	            units = unitsToProcess,  
	            unitsWithVisionInCell = unitsWithVisionInCell,  
	            UnitCount = processQueueIndex,  
	            GridBounds = GridBounds  
	        };  
	        
	        _updateLitCellsJobHandle = job.Schedule();  
	        _jobRunning = true;  
	    }
	}  
	  
	private void LateUpdate()  
	{  
	    if (!_jobRunning) return;
	      
	    _updateLitCellsJobHandle.Complete();  
	    _jobRunning = false;  
	}

}
```

### The Burst Compiler

To use the burst compiler, we just need to add the attribute [BurstCompile].

To test the speed of using the Burst compiler I turned off the optimisation to skip units that hadn't moved, and completed the job immediately to more easily profile how long it was taking.

I added around 20-30 units to the scene, and the above was taking around 0.5ms.

After adding the Burst compiler that went to 0.03ms, a 17x improvement.

```c#
[BurstCompile]  
public struct UpdateUnitVisibilityJob : IJob  
{
	 //...
}

```

Even without the further improvement of ignoring cells that a unit will remain in, the calculation is now having negligible impact on performance.


**More Improvements**

We are still iterating over all units on the main thread to decide if they have moved and thus need to update the vision map. This step could also be made into a job, or even better the units could be converted to Entities and make use of Unity's ECS system.

### Updating the Texture

To make use of the unit visibility info, we want to write the data into a texture, which can then be used by a shader. Writing to the texture can also be done in a job by creating a NativeArray using '.GetRawTextureData()'.

On this same texture, we are using the red channel to determine which areas are buildable.

```c#
[BurstCompile]  
public struct UpdateGridTextureJob : IJobParallelFor  
{  
    [ReadOnly] public NativeArray<bool> blockedCells;  
    [ReadOnly] public NativeArray<uint> unitsWithVisionInCell;  
    [WriteOnly] public NativeArray<Color32> texture;  
    
    public void Execute(int index)  
    {        
	    byte r = blockedCells[index] ? byte.MaxValue : byte.MinValue;  
        byte g = byte.MinValue;  
        byte b = unitsWithVisionInCell[index] > 0 ? byte.MaxValue : byte.MinValue; 
        byte a = byte.MaxValue;  
        
        texture[index] = new Color32(r, g, b, a);  
    }
}
```


Because this job depends on the previous, we pass in the job handle '\_updateLitCellsJobHandle' when scheduling.

```c#

[SerializeField] private Texture2D gridTexture;
private NativeArray<Color32> gridTextureData;

private void Start()
{
	// define the texture data
	gridTextureData = gridTexture.GetRawTextureData<Color32>();
}

//...

private void Update()
{
	// ...
	
	// Schedule job after the calculations are finished
	var updateTextureJob = new UpdateGridTextureJob()  
	{  
	    blockedCells = blockedCells,  
	    unitsWithVisionInCell = unitsWithVisionInCell,  
	    texture = gridTextureData  
	};  
	_updateTextureJobHandle = updateTextureJob.Schedule(GridBounds*GridBounds, 8, _updateLitCellsJobHandle);
}
//...

private void LateUpdate()
{
	// When job finished, apply the texture
	_updateTextureJobHandle.Complete();
	gridTexture.Apply();
}
```

## Unit Attack Target Calculations
