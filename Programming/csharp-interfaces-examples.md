# C# Interfaces — Quick Examples

[← Back to index](README.md)

## Basic interface + implementation
```csharp
public interface IShape
{
    double Area { get; }
    void Draw();
}

public class Circle : IShape
{
    public double Radius { get; set; }
    public double Area => Math.PI * Radius * Radius;
    public void Draw() => Console.WriteLine("Circle drawn");
}
```

## Multiple interfaces on one class
```csharp
public interface IMovable { void Move(); }
public interface IDamageable { void TakeDamage(int amount); }

public class Player : IMovable, IDamageable
{
    public void Move() => Console.WriteLine("Moving");
    public void TakeDamage(int amount) => Console.WriteLine($"Took {amount} damage");
}
```

## Interface inheriting another interface
```csharp
public interface IAnimal { void Eat(); }
public interface IPet : IAnimal { void Play(); }

public class Dog : IPet
{
    public void Eat() => Console.WriteLine("Eating");
    public void Play() => Console.WriteLine("Playing");
}
```

## Explicit interface implementation (name collision)
```csharp
public interface IA { void DoWork(); }
public interface IB { void DoWork(); }

public class Foo : IA, IB
{
    void IA.DoWork() => Console.WriteLine("IA version");
    void IB.DoWork() => Console.WriteLine("IB version");
}

// Usage:
Foo foo = new Foo();
((IA)foo).DoWork(); // "IA version"
((IB)foo).DoWork(); // "IB version"
```

## Default interface method (C# 8+)
```csharp
public interface ILogger
{
    void Log(string message);
    void LogError(string message) => Log($"ERROR: {message}"); // no override needed
}

public class ConsoleLogger : ILogger
{
    public void Log(string message) => Console.WriteLine(message);
    // LogError inherited automatically
}
```

## Interface vs abstract class, side by side
```csharp
// Interface: contract only, no state
public interface IEngine
{
    void Start();
}

// Abstract class: shared state + partial implementation
public abstract class VehicleBase
{
    public int Speed { get; protected set; }
    public abstract void Accelerate();
}
```

## Runtime type check with `is`
```csharp
IShape shape = new Circle { Radius = 5 };

if (shape is Circle circle)
{
    Console.WriteLine($"It's a circle with radius {circle.Radius}");
}
```

## Polymorphism via interface list
```csharp
List<IShape> shapes = new List<IShape> { new Circle { Radius = 2 } };

foreach (var s in shapes)
{
    s.Draw(); // calls the correct implementation for each type
}
```

## Built-in interface example — `IComparable<T>`
```csharp
public class Player : IComparable<Player>
{
    public int Score { get; set; }

    public int CompareTo(Player other) => Score.CompareTo(other.Score);
}

// Enables:
players.Sort();
```
