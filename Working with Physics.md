# Working with Physics

## Summary
At first glimpse the units of the properties and values concerning physics on Roblox 
are magical inventions complying to irrational rules documented in hushed tones around 
the elite developer circles, but this couldn't be further from the truth.
The Roblox physics engine adheres to the newtonian physics model, just at
a rather curious scale. Knowing the scaled conversions between real life 
representation and Roblox's representation of physics units can prevent
a plethora of common issues, such as system over constraint, unexpected/unrealistic 
results, and the frantic runaway of parts in every direction as soon as the system is unanchored.

## Unit Conversion
To start understanding Roblox's units we first look at Workspace.Gravity. 
Default this value is 196.200; a factor of 9.81 by 20. Given the gravitational
acceleration of 9.81 m/s/s we can determine that 1 stud is represented as 5 centimeters.
Given the mass of 8000 from a 20,20,20 size part representing a cubic meter with density set to 1. A 1 cubic meter
of water can be assumed to have a mass of 1000kg based on 1 cubic cm of water having a mass of 1 gram.
This gives us the relation of 1000kg = 8000 roblox "masses"

To recap:
 * 1 stud = 5cm 
 * 1 mass = 0.125kg

## Simple Conversion Functions

```lua
local function cmToStuds(cmDist)
   return cmDist * .2
end

local function studsToCm(studsDist)
    return studsDist * 5
end

local function kgToRMass(kgMass)
    return kgMass * 8
end

local function rMassToKg(rMass)
    return rMass * 0.125
end
```
