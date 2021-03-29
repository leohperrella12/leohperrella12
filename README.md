import random
import pickle
import os
import struct

INVENTORY = 1001
MINIMUM_ID_FOR_ITEM = 2001
ID_DIFFERENCE_FOR_OBJECT_IN_TWO_LOCATIONS = 10000

class Place():
  def __init__(self):
    self.Description = ""
    self.ID = self.North = self.East = self.South = self.West = self.Up = self.Down = 0

class Character():
  def __init__(self):
    self.Name = self.Description = ""
    self.ID = self.CurrentLocation = 0

class Item():
  def __init__(self):
    self.ID = self.Location = 0
    self.Description = self.Status = self.Name = self.Commands = self.Results = ""

def GetInstruction():
  print(os.linesep)
  Instruction = input("> ").lower()
  return Instruction

def ExtractCommand(Instruction):
  Command = ""
  if " " not in Instruction:
    return Instruction, Instruction
  while len(Instruction) > 0 and Instruction[0] != " ":
    Command += Instruction[0]
    Instruction = Instruction[1:]
  while len(Instruction) > 0 and Instruction[0] == " ":
    Instruction = Instruction[1:]
  return Command, Instruction

def Go(You, Direction, CurrentPlace):
  Moved = True
  print("Currentplace is ",You.CurrentLocation-1)
  print("Place ",CurrentPlace.North,CurrentPlace.East,CurrentPlace.South,CurrentPlace.West,CurrentPlace.Up,CurrentPlace.Down)
  if Direction == "north":
    if CurrentPlace.North == 0:
      Moved = False
    else:
      You.CurrentLocation = CurrentPlace.North
  elif Direction == "east":
    if CurrentPlace.East == 0:
      Moved = False
    else:
      You.CurrentLocation = CurrentPlace.East
  elif Direction == "south":
    if CurrentPlace.South == 0:
      Moved = False
    else:
      You.CurrentLocation = CurrentPlace.South
  elif Direction == "west":
    if CurrentPlace.West == 0:
      Moved = False
    else:
      You.CurrentLocation = CurrentPlace.West
  elif Direction == "up":
    if CurrentPlace.Up == 0:
      Moved = False
    else:
      You.CurrentLocation = CurrentPlace.Up
  elif Direction == "down":
    if CurrentPlace.Down == 0:
      Moved = False
    else:
      You.CurrentLocation = CurrentPlace.Down
  else:
    Moved = False
  if not Moved:
    print("You are not able to go in that direction.")
  print("Currentplace is now ",You.CurrentLocation-1)
  return You, Moved

def DisplayDoorStatus(Status):
  if Status == "open":
    print("The door is open.")
  else:
    print("The door is closed.")

def DisplayContentsOfContainerItem(Items, ContainerID):
  print("It contains: ", end = "")
  ContainsItem = False
  for Thing in Items:
    if Thing.Location == ContainerID:
      if ContainsItem:
        print(", ", end = "")
      ContainsItem = True
      print(Thing.Name, end = "")
  if ContainsItem:
    print(".")
  else:
    print("nothing.")

def Examine(Items, Characters, ItemToExamine, CurrentLocation):
  Count = 0
  if ItemToExamine == "inventory":
    DisplayInventory(Items)
  else:
    IndexOfItem = GetIndexOfItem(ItemToExamine, -1, Items)
    if IndexOfItem != -1:
      if Items[IndexOfItem].Location == INVENTORY or Items[IndexOfItem].Location == CurrentLocation:
        print(Items[IndexOfItem].Description)
        if "door" in Items[IndexOfItem].Name:
          DisplayDoorStatus(Items[IndexOfItem].Status)
        if "container" in Items[IndexOfItem].Status:
          DisplayContentsOfContainerItem(Items, Items[IndexOfItem].ID)
        return
    while Count < len(Characters):
      if Characters[Count].Name == ItemToExamine and Characters[Count].CurrentLocation == CurrentLocation:
        print(Characters[Count].Description)
        return
      Count += 1
    print("You cannot find " + ItemToExamine + " to look at.")

def GetPositionOfCommand(CommandList, Command):
  # Find position of the input command in the command list
  Position = Count = 0
  while Count <= len(CommandList) - len(Command):
    # There are enough letters left in the command list for this command to be there
    if CommandList[Count:Count + len(Command)] == Command:
      # This is the required command
      return Position
    elif CommandList[Count] == ",":
      # Found a comma, so the command is later in the list
      Position += 1
    # Move to the next character
    Count += 1
  return Position

def GetResultForCommand(Results, Position):
  # Find the result for the input position
  # Results are separated by semi-colons
  Count = 0
  CurrentPosition = 0
  ResultForCommand = ""
  while CurrentPosition < Position and Count < len(Results):
    # Not yet reached the required result
    if Results[Count] == ";":
      # Next result
      CurrentPosition += 1
    Count += 1
  while Count < len(Results):
    # Add all the characters that make up the result
    if Results[Count] == ";":
      # Text terminates at the semi-colon
      break
    ResultForCommand += Results[Count]
    Count += 1
  return ResultForCommand

def Say(Speech):
  print()
  print(Speech)
  print()

def ExtractResultForCommand(SubCommand, SubCommandParameter, ResultForCommand):
  # Extract the first two components for this result
  # e.g   "say,It is attached to the wall"
  # produces Subcommand="say" and subcommandparameter="It is attached to the wall"
  # Note that the outputs are appended to in this routine.
  Count = 0
  while Count < len(ResultForCommand) and ResultForCommand[Count] != ",":
    # Not yet reached the comma or the end of this result
    SubCommand += ResultForCommand[Count]
    Count += 1
  # Remainder of result are parameters - read until end of result or until a comma or semi-colon is reached
  Count += 1
  while Count < len(ResultForCommand):
    if ResultForCommand[Count] != "," and ResultForCommand[Count] != ";":
      # Not a comma or a semi-colon - add to string
      SubCommandParameter += ResultForCommand[Count]
    else:
      # found a comma or semi-colon - terminate
      break
    Count += 1
  return SubCommand, SubCommandParameter

def ChangeLocationReference(Direction, NewLocationReference, Places, IndexOfCurrentLocation, Opposite):
  # Next line is DEAD CODE that wastes memory
  ThisPlace = Place()
  ThisPlace = Places[IndexOfCurrentLocation]
  # It is better to have brackets in the following conditions as the "and" must be
  # applied before the "or"
  if (Direction == "north" and not Opposite) or (Direction == "south" and Opposite):
    # Direction is north or the opposite of south
    ThisPlace.North = NewLocationReference
  elif Direction == "east" and not Opposite or Direction == "west" and Opposite:
    ThisPlace.East = NewLocationReference
  elif Direction == "south" and not Opposite or Direction == "north" and Opposite:
    ThisPlace.South = NewLocationReference
  elif Direction == "west" and not Opposite or Direction == "east" and Opposite:
    ThisPlace.West = NewLocationReference
  elif Direction == "up" and not Opposite or Direction == "down" and Opposite:
    ThisPlace.Up = NewLocationReference
  elif Direction == "down" and not Opposite or Direction == "up" and Opposite:
    ThisPlace.Down = NewLocationReference
  # Next line is pointless as this is already the case (as they already point to the same place)
  Places[IndexOfCurrentLocation] = ThisPlace
  return Places

def OpenClose(Open, Items, Places, ItemToOpenClose, CurrentLocation):
  # Open or close a door
  Count = 0
  Direction = ""
