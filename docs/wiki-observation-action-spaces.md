# Observation & Action Spaces

## Overview

The success of multi-agent reinforcement learning depends on well-designed observation and action spaces. EcoSculptor implements domain-specific spaces that encourage emergent ecological behaviors.

---

## Observation Space

### 1. Ray Perception Sensor

**Purpose**: Allow agents to sense nearby entities without explicit global knowledge

#### Configuration

```csharp
// Ray casting in multiple directions
public class RayPerceptionSensorComponent : SensorComponent
{
    public int rayCount = 8;              // Number of rays per agent
    public float rayLength = 30f;         // Maximum detection range
    public int layerMask = ~0;            // All physical layers
    public string[] detectableObjects;    // Tags to detect
}
```

#### How It Works

```
Agent Position
     |
     ↓
   /|\\\        ← 8 rays cast outward
  / | \ \       
 /  |  \ \      Each ray reports:
/   |   \ \     • Distance to nearest object
\   |   / /     • Type of object (tag)
 \  |  / /      • Confidence score
  \ | / /
   \|/
```

#### Detected Objects

| Tag | Meaning | Behavior |
|-----|---------|----------|
| `"nectar"` | Food source | Prey seeks; others ignore |
| `"Agent"` | Prey animal | Hunters target; prey avoid |
| `"Hunter"` | Hunter agent | Alpha targets; others avoid |
| `"boundary"` | Environmental limit | All agents avoid (implicit) |

#### Ray Output Format

For each ray:
```
[distance_to_object, object_type_id, ...]
```

**8 rays × (distance + type) = ~16 input features from ray sensor**

---

### 2. Vector Observation

**Purpose**: Agent self-awareness of absolute position

#### Implementation

```csharp
public override void CollectObservations(VectorSensor sensor)
{
    // Own position in world space
    sensor.AddObservation(transform.localPosition);  // 3 floats: (x, y, z)
}
```

#### Position Space

- **X-axis**: [-50, 50] (horizontal spread)
- **Y-axis**: [0.4, 0.4] (fixed height, agents don't fly)
- **Z-axis**: [-50, 50] (horizontal spread)

**Normalized range**: Position values divided by map size for networks

#### Why Include Position?

- Allows agents to learn spatial strategies
- Prevents clustering (spread across map)
- Encodes implicit "I'm near boundary" information

---

### 3. Complete Observation Vector

```
┌──────────────────────────────────────────────────┐
│ Ray Perception Sensor Outputs                    │
│ ├─ Ray 1: [distance, type]                       │
│ ├─ Ray 2: [distance, type]                       │
│ ├─ ...                                           │
│ └─ Ray 8: [distance, type]                       │
│ Total: ~16-24 features                           │
│                                                  │
│ Vector Observation                               │
│ ├─ Position X                                    │
│ ├─ Position Y                                    │
│ └─ Position Z                                    │
│ Total: 3 features                                │
└──────────────────────────────────────────────────┘
        ↓
    Total: ~20-30 input features per observation
```

### ML-Agents Configuration

```yaml
observation_specs:
  - name: "RayPerception"
    shape: [24]           # Flattened ray outputs
    
  - name: "VectorObservation"
    shape: [3]            # Position vector
    
# Total observation space dimension: 27
```

---

## Action Space

### Continuous Control (Box Action Space)

Each agent has **2 continuous actions**:

#### Action 1: Rotation

```
Range: [-1.0, 1.0]

Meaning:
  -1.0  →  Turn left (counterclockwise)
   0.0  →  No rotation
  +1.0  →  Turn right (clockwise)

Physics:
  actualRotation = action * rotateSpeed * Time.deltaTime
  rotateSpeed = 6.0 (configurable per agent)
```

#### Action 2: Forward Movement

```
Range: [-1.0, 1.0]

Meaning:
  -1.0  →  Move backward
   0.0  →  Stop
  +1.0  →  Move forward

Physics:
  velocity = transform.forward * action * moveSpeed * Time.deltaTime * 50
  moveSpeed = 4.0 (PreyAnimal)
            = 5.0 (HunterAnimal)  
            = 4.5 (AlphaHunterAnimal)
```

### Action Space Structure

```csharp
public override void OnActionReceived(ActionBuffers actions)
{
    float moveRotate = actions.ContinuousActions[0];   // [-1, 1] rotation
    float moveForward = actions.ContinuousActions[1];  // [-1, 1] movement
    
    // Apply rotation
    transform.Rotate(0f, moveRotate * rotateSpeed, 0f, Space.Self);
    
    // Apply movement
    if (moveForward >= 0) {  // Only forward movement implemented
        var velocity = rb.velocity = transform.forward 
                                   * moveForward 
                                   * moveSpeed 
                                   * Time.deltaTime * 50;
    }
}
```

### Action Space Semantics

| Situation | Expected Action 1 | Expected Action 2 |
|-----------|-------------------|-------------------|
| See food ahead | 0 (straight) | 1 (move forward) |
| Predator on left | 1 (turn right) | 0.5 (flee) |
| Obstacle ahead | ±1 (turn away) | 0.5 (move around) |
| Safe wandering | ±0.5 (meander) | 0.7 (cruise) |

---

## Agent-Specific Observations

### PreyAnimal Senses

```
High Priority:
  • Food location (ray detection)
  • Threat detection (hunters)
  
Medium Priority:
  • Own position (avoid boundary)
  • Movement efficiency (speed feedback)
  
Low Priority:
  • Other prey (not directly relevant)
```

**Learned Strategy**: Find food, avoid predators

### HunterAnimal Senses

```
High Priority:
  • Prey location (ray detection)
  • Alpha hunter detection (threat)
  
Medium Priority:
  • Own position
  • Other hunters (food competitors?)
  
Low Priority:
  • Food sources (irrelevant)
```

**Learned Strategy**: Hunt prey, avoid stronger predators

### AlphaHunterAnimal Senses

```
High Priority:
  • Multiple prey types (both prey and hunters)
  • Own dominance (highest in chain)
  
Medium Priority:
  • Own position
  • Other alphas (if multiple present)
  
Low Priority:
  • Resources (abundant)
```

**Learned Strategy**: Dominate ecosystem, hunt opportunistically

---

## Sensor Processing Pipeline

```
Raw Environment State
        ↓
  ┌─────────────────────────────────┐
  │ Ray Perception Sensor           │
  │ (Cast 8 rays, detect objects)   │
  └─────────────────────────────────┘
        ↓
  ┌─────────────────────────────────┐
  │ Ray Results Encoding            │
  │ (Distance + Type → floats)      │
  └─────────────────────────────────┘
        ↓
  ┌─────────────────────────────────┐
  │ Vector Observations             │
  │ (Position + hunger + ...)       │
  └─────────────────────────────────┘
        ↓
  ┌─────────────────────────────────┐
  │ Flatten & Stack                 │
  │ [ray_features..., position...]  │
  └─────────────────────────────────┘
        ↓
  ┌─────────────────────────────────┐
  │ Neural Network Input            │
  │ Shape: (batch, 27)              │
  └─────────────────────────────────┘
```

---

## Action Execution Pipeline

```
Neural Network Output
        ↓
  ┌─────────────────────────────────┐
  │ Policy Head (μ, σ)              │
  │ Output mean & std for actions   │
  └─────────────────────────────────┘
        ↓
  ┌─────────────────────────────────┐
  │ Sample from Gaussian            │
  │ a ~ N(μ, σ²)                   │
  │ Clamp to [-1, 1]                │
  └─────────────────────────────────┘
        ↓
  ┌─────────────────────────────────┐
  │ Denormalize Actions             │
  │ rotation = a₀ * rotateSpeed     │
  │ velocity = a₁ * moveSpeed       │
  └─────────────────────────────────┘
        ↓
  ┌─────────────────────────────────┐
  │ Apply to Agent                  │
  │ • Update rotation               │
  │ • Update velocity               │
  │ • Trigger animations            │
  └─────────────────────────────────┘
        ↓
  Environment State Changes
```

---

## Information Content Analysis

### Observation Space Dimensionality

| Component | Features | Purpose |
|-----------|----------|---------|
| Ray Perception (8 rays) | 16-24 | Spatial awareness |
| Position Vector | 3 | Self-localization |
| **Total** | **~27** | **Compact yet expressive** |

### Action Space Dimensionality

| Component | Dimension | Values |
|-----------|-----------|--------|
| Rotation | 1 | Continuous [-1, 1] |
| Movement | 1 | Continuous [-1, 1] |
| **Total** | **2** | **Simple, reactive control** |

### Information Capacity

- **Observation entropy**: Limited to local (~30m radius) perception
- **Action entropy**: Simple 2D control; no jumping, flying, or attacks
- **Trade-off**: Agents must learn to solve complex problems with simple primitives

---

## Design Rationale

### Why These Specific Choices?

#### ✅ Ray Perception (vs. Grid/Image Observations)

**Advantages**:
- Efficient (not processing entire map)
- Interpretable (we can see what agents detect)
- Scalable (works with any map size)
- Biology-inspired (animals use directional sensing)

**Disadvantages**:
- Limited to detecting tagged objects
- No visual details of environment

#### ✅ Continuous Actions (vs. Discrete)

**Advantages**:
- Smooth, natural movement
- Agents learn speed control
- Realistic physics-based behavior
- No discretization artifacts

**Disadvantages**:
- Harder for networks to learn
- More sample-inefficient
- Requires careful scaling

#### ✅ Vector Observations (vs. No Position)

**Advantages**:
- Prevents agent clustering
- Allows boundary awareness
- Self-localization improves learning

**Disadvantages**:
- Gives agents "global" information
- Breaks strict locality

---

## Sensor Debugging

### Visualization in Unity

```csharp
// Debug visualization of ray perception
void OnDrawGizmosSelected()
{
    foreach (var ray in rayResults) {
        Gizmos.color = ray.objectHit ? Color.red : Color.green;
        Gizmos.DrawRay(transform.position, ray.direction * ray.distance);
    }
}
```

### What Good Observations Enable

✓ Agents learn efficient pathfinding
✓ Predators develop hunting strategies  
✓ Prey develop evasion behaviors
✓ Emergent coordination without explicit rules

---

## Future Improvements

### Potential Observation Enhancements

1. **Hunger Level**: Add agent's current hunger to observations
2. **Velocity Vector**: Include own velocity for momentum awareness
3. **Health/Energy**: Explicit fitness indicator
4. **Pheromone Trails**: Stigmergy for collective behavior

### Potential Action Enhancements

1. **Attack/Bite**: Discrete action for hunting
2. **Eat**: Action to consume food when on it
3. **Rest**: Strategic stamina management
4. **Vocalization**: Communication between agents

---

## Learning from Observations

### What Emerges

- **Prey**: Learns to flee when predator detected nearby
- **Hunters**: Learns to chase when prey detected
- **Alpha**: Learns to intercept both prey and hunters

### Information Flow

```
Observation → Neural Network → Action → Environment Change → New Observation
   (state)                      (policy)    (transition)        (next state)
     ↑____________________________________________________________________________________↑
```

The agent improves its policy by maximizing expected return over this loop.

