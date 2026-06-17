# Contribution Log<!-- omit in toc -->

- [Contribution #1: Animated sprite for sprite component](#contribution-1-animated-sprite-for-sprite-component)
  - [Why I Chose This Issue](#why-i-chose-this-issue)
  - [Understanding the Issue](#understanding-the-issue)
  - [Reproduction Process](#reproduction-process)
  - [Solution Approach](#solution-approach)
  - [Testing Strategy](#testing-strategy)
  - [Implementation Notes](#implementation-notes)
  - [Pull Request](#pull-request)
  - [Learnings \& Reflections](#learnings--reflections)
  - [Resources Used](#resources-used)

## Contribution #1: Animated sprite for sprite component

**Contribution Number:** 1  
**Student:** Evan Fish  
**Issue:** [[GitHub issue link]](https://github.com/ezEngine/ezEngine/issues/1706)  
**Status:** Phase I Complete

---

### Why I Chose This Issue

This issue interests me as I have explored and attempted game engine development beforehand. I find the low-level constraints interesting and very engaging. This issue works within a game engine, and specifically adresses the sprite component that is within the engine. Which caught my eye as I have had previous experience working with sprites, and I think that I can bring a interesting perspective to the feature.

---

### Understanding the Issue

#### Problem Description

[In your own words, what's broken or missing?]

#### Expected Behavior

[What should happen?]

#### Current Behavior

[What actually happens?]

#### Affected Components

[Which parts of the codebase are involved?]

---

### Reproduction Process

#### Environment Setup

Initialized the project using Visual Studio 2026. Longest part of setup was installing Visual Studio (the download webpage was broken).

Working branch: <https://github.com/Mr-Fishy/ezEngine-sprite-component/tree/animated-sprite>

#### Steps to Reproduce

1. Create an `ezSpriteComponent` and assign a spritesheet texture (e.g., an 8x8 grid of an explosion animation).
2. Observe the rendered output in the scene.
3. **Observed result:** The entire spritesheet is rendered on a single quad, static and unmoving. There is no way to define grid dimensions, advance frames, or disable the maximum screen size constraint.

#### Reproduction Evidence

- **Commit showing reproduction:** N/A - Feature Implementation
- **Screenshots/logs:** N/A
- **My findings:** The `ezSpriteRenderData` struct already contains `m_textCoordScale` and `m_texCoordOffset`. However, in `ezSpriteComponent::OnMsgExtractRenderData`, these are currently hardcoded to `ezVec2(1.0f)` and `ezVec2(0.0f)`. These confirm that the rendering pipeline is already equipped to handle sub-texture rendering, so the work is isolated to the component logic and UI reflection.

---

### Solution Approach

#### Analysis

To support spritesheet animations, the component needs to track time and calculate the correct UV scale and offset per frame. Because the animation end behavior is requested to be an option, the component must be able to actively process time and potentially destroy its owner. Currently, `ezSpriteComponentManager` is a standard compact manager with no update tick. It needs to be converted into a ticking component manager (like `ezComponentManagerSimple`) to process frame time.

For the optional "Max Screen Size", adding a boolean toggle to the component is the cleanest approach. When disabled, we can pass a massive float value (like `ezMath::HighValue<float>()`) to the render data, effectively bypassing the shader's size clamp without touching the rendering pipeline.

#### Proposed Solution

1. **Component State:** Add properties for the grid (`Columns`, `Rows`), animation settings (`Framerate`, `Loop`, `MaxLoops`, `EndAction`), and a toggle for `UseMaxScreenSize`.
2. **Time Processing:** Upgrade the component manager to tick every frame so it can accumulate "sprite time" and handle the `Delete` end action via `GetWorld()->DeleteObjectNow()`.
3. **UV Math:** In `OnMsgExtractRenderData`, dynamically calculate `m_textCoordScale` based on the gride, and `m_textCoordOffset` based on the current active frame index.

#### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** We need to augment `ezSpriteComponent` to support spritesheet animation, looping behaviors, and an optional screen size clamp, utilizing the existing UV scaling properties in the render pipeline.

**Match:** This behaves similarly to other simple animated components or particle effects in ezEngine. I will use `ezComponentManagerSimple` to gain an `Update()` tick, and `ezMath` for UV offset calculations. I will create a new reflectable enum for the end action (Restart, Delete, Nothing).

**Plan:**

1. Modify `SpriteComopnent.h`
   - Define a new enum: `ezSpriteAnimationEndAction` (Restart, Delete, Nothing).
   - Update the `ezSpriteComponentManager` typedef:

    ```cpp
    using ezSpriteComponentManager = ezComponentManagerSimple<class ezSpriteComponent, ezComponentUpdateType::Always, ezBlockStorageType::Compact>;
    ```

   - Add new properties to `ezSpriteComponent`: `m_uiColumns`, `m_uiRows`, `m_uiTotalSprites`, `m_fFramerate`, `m_bLoop`, `m_iMaxLoops`, `m_EndAction`, and `m_bUseMaxScreenSize`.
   - Add internal state variables: `m_TimeSinceStart` (ezTime), `m_uiCurrentLoop` (ezUInt32), and `m_bIsFinished` (bool).
   - Add and `Update()` method for the manager to call.
2. Modify `SpriteComponent.cpp`
   - Register the new enum using `EZ_BEGIN_STATIC_REFLECTED_ENUM`.
   - Add the new properties to `EZ_BEGIN_COMPONENT_TYPE` block inside `EZ_BEGIN_PROPERTIES`. Use appropriate default attributes (e.g., columns/rows = 1, Loop = true, MaxLoops = -1).
   - Bump the component version from `3` to `4` in `EZ_BEGIN_COMPONENT_TYPE`.
   - Update `SerializeComponent` and `DeserializeComponent` to read / write the new version 4 variables.
3. Implement logic in `SpriteComponent.cpp`
   - In `Update()`
     - Accumulate `GetWorld()->GetClock()->GetTimeDiff()` into `m_TimeSinceStart`.
     - Calculate current frame. If `m_uiCurrentLoop > m_iMaxLoops` (and `m_iMaxLoops != -1`), trigger the `m_EndAction`.
     - If `m_EndAction == Delete`, call `GetWorld()->DeleteObjectNow(GetOwner()->GetHandle())`.
   - In `OnMsgExtractRenderData()`
     - If `!m_bUseMaxScreenSize`, set `pRenderData->m_fMaxScreenSize = ezMath::HighValue<float>()`.
     - Calculate scale: `pRenderData->m_textCoordScale = ezVec2(1.0f / m_uiColumns, 1.0f / m_uiRows)`.
     - Calculate current frame index based on `m_TimeSinceStart` and `m_fFramerate`.
     - Calculate offset:

      ```cpp
      ezUInt32 x = frameIndex % m_uiColumns;
      ezUInt32 y = frameIndex / m_uiColumns;
      pRenderData->m_texCoordOffset = ezVec2((float)x * pRenderData->m_texCoordScale.x, (float)y * pRenderData->m_texCoordScale.y);
      ```

**Implement:** [Link to your branch/commits as you work]

**Review:** [Self-review checklist - does it follow the project's contribution guidelines?]

**Evaluate:** I will verify this works by loading an existing sample scene with a static sprite (to ensure backwards compatibility works), then creating a new entity with a 4x4 explosion spritesheet. I will test the loop toggle, ensure the max loops limits playback, and confirm the entity vanishes when set to `Delete`. Finally, I will disable `MaxScreenSize` and walk the camera backwards to ensure the sprite shrinks infinitely instead of clamping.

---

### Testing Strategy

#### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

#### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

#### Manual Testing

[What you tested manually and results]

---

### Implementation Notes

#### Week [X] Progress

[What you built this week, challenges faced, decisions made]

#### Week [Y] Progress

[Continue documenting as you work]

#### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

### Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**

- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

### Learnings & Reflections

#### Technical Skills Gained

[What you learned technically]

#### Challenges Overcome

[What was hard and how you solved it]

#### What I'd Do Differently Next Time

[Reflection on your process]

---

### Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
