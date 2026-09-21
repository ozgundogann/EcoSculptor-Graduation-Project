# EcoSculptor

**A god-view ecosystem simulation in which reinforcement-learning animals react to the terrain the player builds.**

[▶ Gameplay video](https://youtu.be/ViMa-MOm5ZM) · [Wiki](https://github.com/ozgundogann/EcoSculptor-Graduation-Project/wiki) · Unity 2021.3.16f1 · ML-Agents · PPO

EcoSculptor is a single-player Unity game. The player rebuilds a hex-tile world (rivers, grass, sand, dirt, stone) and, through the terrain, decides which animals can survive there. The animals are not scripted: prey, hunters and alpha hunters are neural-network policies trained with Proximal Policy Optimization in the Unity ML-Agents Toolkit. It was developed as a two-year graduation project (CENG 407/408, Çankaya University, 2023–2024).

## What is in it

- **Terrain-driven ecosystem.** Tile counts trigger animal spawning (rivers → deer, dirt → horses, sand → tigers, and so on). Food grows only on grass beside rivers, and hunger, a day/night cycle and seasons add pressure.
- **Three learned policies for five species.** Deer and horse share a prey policy, wolf and tiger share a hunter policy, and the bear is the alpha hunter, which also hunts hunters.
- **Multi-agent training.** The three policies are trained simultaneously in one arena with shared episode termination, competitive and without self-play.
- **Documented learning problem.** Observations (ray perception + position, 208 / 167 / 208 inputs), actions (turn, forward), reward table, hyperparameters and training history are written up in the wiki.

| Behavior | Species | Inputs | Actions |
|---|---|---|---|
| `Preyanimal` | Deer, Horse | 208 | 2 continuous |
| `Hunteranimal` | Wolf, Tiger | 167 | 2 continuous |
| `Alphahunteranimal` | Bear | 208 | 2 continuous |

## Training result

![Mean cumulative reward per episode in the logged run Final_1.3](docs/images/training-reward-final-1-3.svg)

In the last complete three-agent run (`Final_1.3`, 2 M steps per behavior), the prey policy improved clearly and the alpha hunter improved and then plateaued, while the wolf/tiger policy showed no clear improvement. The deployed models (`Assets/Trainings/4.0`) come from a later run whose logs were not preserved. The [Results](https://github.com/ozgundogann/EcoSculptor-Graduation-Project/wiki/Results) and [Limitations](https://github.com/ozgundogann/EcoSculptor-Graduation-Project/wiki/Limitations-and-Future-Work) pages state exactly what the figure does and does not show.

## Quick start

1. Clone the repository and open it with **Unity 2021.3.16f1** (Unity restores the packages).
2. Open `Assets/Scenes/Main Menu.unity`, press **Play**, and choose **New Game**.

The animals run their trained policies (`Assets/Trainings/4.0`) with no Python needed. To retrain, see [Getting Started](https://github.com/ozgundogann/EcoSculptor-Graduation-Project/wiki/Getting-Started) and [Training](https://github.com/ozgundogann/EcoSculptor-Graduation-Project/wiki/Training).

## Repository layout

| Path | Contents |
|---|---|
| `Assets/Scripts/` | Game and agent code (`Animals`, `Tiles`, `Managers`, `Economy`, `Camera`, `Day&Night Cycle`) |
| `Assets/Prefabs/Animals/` | Species prefabs with Behavior Parameters and ray sensors |
| `Assets/Trainings/` | Exported `.onnx` policies (`4.0` is the deployed set) |
| `results/` | ML-Agents runs: checkpoints, configuration, TensorBoard logs |
| `MultiTraining.yaml` | A trainer configuration (see the Training page for which models it belongs to) |

## Documentation

| | |
|---|---|
| [Gameplay and Ecosystem Design](https://github.com/ozgundogann/EcoSculptor-Graduation-Project/wiki/Gameplay-and-Ecosystem-Design) | Rules, terrain, economy, controls |
| [System Architecture](https://github.com/ozgundogann/EcoSculptor-Graduation-Project/wiki/System-Architecture) | Subsystems and how they connect |
| [Agents and Environment](https://github.com/ozgundogann/EcoSculptor-Graduation-Project/wiki/Agents-and-Environment) | Observations, actions, rewards, training arena |
| [Training](https://github.com/ozgundogann/EcoSculptor-Graduation-Project/wiki/Training) | PPO settings, run history, model archive |
| [Results](https://github.com/ozgundogann/EcoSculptor-Graduation-Project/wiki/Results) | Learning curves and threats to validity |
| [Limitations and Future Work](https://github.com/ozgundogann/EcoSculptor-Graduation-Project/wiki/Limitations-and-Future-Work) | What is missing or unverified |
| [Archive](https://github.com/ozgundogann/EcoSculptor-Graduation-Project/wiki/Archive) | Original course documents (SRS, GDD, test plan, report) |

## Team

Özgün Doğan, Doğa Melis Erke, Ege Özçelik and Emre Erişen. Advisor: Dr. Abdül Kadir Görür. Special thanks to Cenk Ündan for the assets and animations. This is a public copy of the team's [original course repository](https://github.com/CankayaUniversity/ceng-407-408-2023-2024-EcoSculptor). See [Team and Credits](https://github.com/ozgundogann/EcoSculptor-Graduation-Project/wiki/Team-and-Credits) for roles and third-party components.
