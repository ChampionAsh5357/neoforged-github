# Minecraft 26.3 -> 26.4 Mod Migration Primer

This is a high level, non-exhaustive overview on how to migrate your mod from 26.3 to 26.4. This does not look at any specific mod loader, just the changes to the vanilla classes. All provided names use the official mojang mappings. You can find the [changelog here](./changelog.md).

This primer is licensed under the [Creative Commons Attribution 4.0 International](http://creativecommons.org/licenses/by/4.0/), so feel free to use it as a reference and leave a link so that other readers can consume the primer.

If there's any incorrect or missing information, please file an issue on this repository or ping @ChampionAsh5357 in the Neoforged Discord server.

Thank you to:

- @bl4ckscor3 for `BY_ID` within enums now replaced by `EnumStreamCodec#byId`

## Pack Changes

There are a number of user-facing changes that are part of vanilla which are not discussed below that may be relevant to modders. You can find a list of them on [Misode's version changelog](https://misode.github.io/versions/?id=26.4&tab=changelog).

## Client Registries

Vanilla has now formalized the concept of a registration system for client-only objects, similar to normal registration via `BuiltInRegistries`. The client registry keys are stored in `ClientRegistries`, while the registries themselves are in `BuiltInClientRegistries`. Currently, the registries supported are for `BlockEntityRendererProvider`, `DebugScreenEntry`, `EntityRendererProvider`, `MenuScreens$ScreenConstructor`, and `ParticleProvider`. They are registered using the same `Registry#register` methods.

For `BlockEntityRendererProvider`, `EntityRendererProvider`, `MenuScreens$ScreenConstructor`, and `ParticleProvider`; the associated `ResourceKey` must have an `Identifier` that matches the associate `BlockEntityType`, `EntityType`, `MenuType`, or `ParticleType`; respectively. This is how the vanilla systems know to link the common and client registry objects.

```java
// An example using `MenuType` and `MenuScreens$ScreenConstructor`

// Assume we have some `ExampleMenu` extends `AbstractContainerMenu`
// Assume we have some `ExampleScreen` extends `AbstractContainerScreen`

// The common registry object.
public static final MenuType<ExampleMenu> EXAMPLE_MENU = Registry.register(
    BuiltInRegistries.MENU,
    // The registry object name.
    Identifier.fromNamespaceAndPath("examplemod", "example_menu"),
    // Note that the constructor is currently private and must be made public.
    new MenuType<>(ExampleMenu::new, FeatureFlags.VANILLA_SET)
);

// The client registry object (in some other class).
Registry.register(
    BuiltInClientRegistries.MENU_SCREEN,
    // The registry object name must match the menu type name exactly.
    Identifier.fromNamespaceAndPath("examplemod", "example_menu"),
    ExampleScreen::new
);
```

## Texture Handles and Providers

The way textures are loaded and stored by the `TextureManager` has been rewritten. `AbstractTexture` and its subclass no longer exists. Instead, textures are generally loaded through a `TextureProvider` and then referenced through its loaded `TextureHandle`, splitting the previous implementation. The previous `AbstractTexture` subclasses have corollaries with the `TextureProvider` implementations, as storage remains quite uniform across the implementations.

`TextureProvider`s load the textures via `TextureManager#registerAndLoad` through three methods.

First, `prepareState` is called to read the texture from disk, taking in the `ResourceManager` and `Identifier`, returning the loaded 'file' data represented as a generic. Vanilla uses `TextureContents` here as it holds the `NativeImage` along with the `TextureMetadataSection`. If any exception is thrown (e.g., no file found), then `missingState` is returned instead. The loaded 'file' data must be an `UncheckedAutoCloseable` to release any resources once used.

Then, the `TextureHandle` is created through `createTexture`, taking in the loaded 'file' data and `Identifier`. `createTexture` actually enforces the `TextureHandle` to be a `TextureResources` to support a common dumping and closing implementation. The returned `TextureHandle` exposes the `GpuTextureView` and `GpuSampler` to use in a shader.

By default, textures are loaded through the `TextureProvider2d`; however, a custom texture can be set for an `Identifier` in one of two ways: `registerProvider` to register how to load the texture from disk, or `register` to directly provide the `TextureResources`:

```java
// An example provider implementation.
// A copy of `TextureProvider2d`.
public record ExampleTextureProvider() implements TextureProvider<TextureContents> {

    @Override
    public TextureContents prepareState(ResourceManager resourceManager, Identifier identifier) throws IOException {
        // Loads the texture as the set generic.
        return TextureContents.load(resourceManager, identifier);
    }

    @Override
    public TextureContents missingState() {
        // The missing texture if an exception is thrown.
        return TextureContents.createMissing();
    }

    @Override
    public TextureResources createTexture(TextureContents state, Identifier identifier) {
        // Create the texture and write the image data.
        AddressMode addressMode = state.clamp() ? AddressMode.CLAMP_TO_EDGE : AddressMode.REPEAT;
        FilterMode minMag = state.blur() ? FilterMode.LINEAR : FilterMode.NEAREST;
        GpuSampler sampler = RenderSystem.getSamplerCache().getSampler(addressMode, addressMode, minMag, minMag, false);
        return TextureResources.from2dImage(identifier::toString, state.image(), sampler);
    }
}

// In some client initialization.
Minecraft.getInstance().getTextureManager()
    .registerProvider(
        // Textures are not loaded relative to any location, so need to specify the entire path.
        // Points to: `assets/examplemod/textures/example_texture.png`
        Identifier.fromNamespaceAndPath("examplemod", "textures/example_texture.png"),
        // the provider instance.
        new ExampleTextureProvider()
    );

Minecraft.getInstance().getTextureManager()
    .register(
        // The identifier of the texture, once again non-relative.
        // Points to: `assets/examplemod/textures/atlas/example_atlas.png`
        Identifier.fromNamespaceAndPath("examplemod", "textures/atlas/example_atlas.png"),
        // The texture resources.
        new TextureResources(...)
    );
```

## Yeah, That's Definitely Renderable

`GuiGraphicsExtractor` now provides access to the mouse XY coordinates along with the `DeltaTracker`. As such, `Renderable` no longer takes in these parameters, instead only providing the `GuiGraphicsExtractor`. Additionally, many GUI classes that were not previously `Renderable` now implement the interface.

```java
// In some `Renderable` subtype
@Override
public void extractRenderState(GuiGraphicsExtractor graphics) {
    // The mouse coordinates.
    int mouseX = graphics.mouseX();
    int mouseY = graphics.mouseY();

    // The partial ticks (also known as 'a').
    float partialTick = graphics.partialTicks();

    // ...
}
```

## Dynamic Render Pipeline Construction

Vanilla `RenderPipeline` creation has been reimagined into a more dynamic construction mechanism. As such, most of the existing `RenderPipeline`s are no longer constants, but rather obtained from the cache or constructed lazily using parameters combined with specific methods.

The major feature this change introduces is `RenderPipelines#dynamicPipeline`, which given some parameter object, constructs the `RenderPipeline`. The constructed pipeline is then cached using its parameters and string id as the key, ready to be queried again on the same input. Note that the id in this case doesn't necessarily refer to the `RenderPipeline` identifier itself, but rather a grouping that the parameters are applied to.

```java
// A parameter object.
public record ExampleParameters(boolean value) {}

// The method used to construct the render pipeline.
public static RenderPipeline example(ExampleParameters parameters) {
    return RenderPipelines.dynamicPipeline(
        // The parameters to take in, can be any object, even `null`.
        parameters,
        // The identifier of the pipeline group.
        "examplemod:example",
        // A function to construct a render pipeline from the parameters.
        // In this example, the input would the object we passed in.
        (ExampleParameters params) -> RenderPipeline.builder(GUI_SNIPPET)
            .withLocation(Identifier.withNamespaceAndPath("examplemod", "example_" + Boolean.toString(params.value())))
            .build()
    );
}
```

Vanilla makes use of these methods as pipeline providers, which are functions that typically take in a nullable `OitStage` (where a non-`null` value means the pipeline supports order-independent transparency), and returns a `RenderPipeline`. The pipeline return value can sometimes be nullable depending on if the implementing system validates whether the pipeline can support OIT.

```java
// An example using `SingleQuadParticle$Layer`.
public static final SingleQuadParticle.Layer EXAMPLE_LAYER = new SingleQuadParticle.Layer(
    false,
    TextureAtlas.LOCATION_PARTICLES,
    // A function that takes in the nullable `OitStage` and returns the
    // `RenderPipeline` to use.
    (@Nullable OitStage stage) -> stage == null
        // Return our render pipeline if OIT isn't used since this example doesn't support it.
        ? example(new ExampleParameters(false))
        // Otherwise return null.
        : null
);
```

Vanilla does something similar with their pipelines, with both the parameter records and their accompanying methods available in `RenderPipelines` (e.g., `block` with `$BlockParameters`, `item` with `$ItemParameters`, etc.). For all of the dynamic pipelines vanilla uses, there are precompiled via `RenderPipelines#dynamicPipelinesToPrecompile`. This is handled through `RenderPipelineVariant` and their implementations, which are basically common parameter holders minus the nullable `OitStage`. Its main use is for precompilation, as otherwise its parameters could be added onto the passed parameter object instead, similar to `$SkyParameters`.

```java
// Creating a block pipeline.
RenderPipelines.block(new RenderPipelines.BlockParameters(BlockRenderPipelineVariant.SOLID, null));
```

How the parameters translate to the previous `RenderPipeline` constants is available in the changelog.

As for custom implementations of `RenderPipeline`, these systems may or may not be used. The only major difference are the `OitStage -> RenderPipeline` providers passed around and that most of the associated `RenderPipeline$Snippet`s have been inlined into the construction methods.

## Block Sound Sets

`SoundType` has been replaced with `BlockSoundSet`, a world datapack registry, that provides the common interaction sounds for blocks.

```json5
// A file located at:
// - `data/examplemod/block_sound_set/example_sound.json
{
    // When present, the sound to play when the block is broken.
    // Must be a registered `SoundEvent`.
    "break_sound": "minecraft:block.stone.break",
    // When present, the sound to play when the block landed on
    // after taking damage from falling.
    // Must be a registered `SoundEvent`.
    "fall_sound": "minecraft:block.stone.fall",
    // When present, the sound to play when the block is being hit
    // (e.g., player breaking).
    // Must be a registered `SoundEvent`.
    "hit_sound": "minecraft:block.stone.hit",
    // When present, the sound to play when the block is placed
    // into the world.
    // Must be a registered `SoundEvent`.
    "place_sound": "minecraft:block.stone.place",
    // When present, the sound to play when the block is stepped
    // on by an entity.
    // Must be a registered `SoundEvent`.
    "step_sound": "minecraft:block.stone.step",
    // The pitch to play the block sounds at.
    // Must be a value between [0.00001, 2].
    // Defaults to 1.
    "pitch": 0.5,
    // The volume to play the block sounds.
    // Must be a value between [0.00001, 10].
    // Defaults to 1.
    "volume": 0.5
}
```

The sound set a block uses can be set through `Block$Properties#sound`. If no sound should be played, use `noSound` instead. All blocks default to using `BlockSoundSets#STONE`.

```java
// For some block.
new Block(
    Block.Properties.of()
        // Takes in the `ResourceKey` of the sound set.
        .sound(ResourceKey.create(Registries.BLOCK_SOUND_SET, Identifier.fromNamespaceAndPath("examplemod", "example_sound")))
);
```

## Optional Sounds

Most sound fields across the various datapack registry objects have been marked as `Optional`, meaning that if a sound is not specified, none will play.

## A Singular Fluid

Flowing and source fluids have been combined into a single `Fluid` instance. The source and flowing state are checked via `isSource`. `FlowingFluid` differentiates source and flowing through the `BlockStateProperties#SOURCE` boolean property. `isFalling` is also added for fluids flowing downwards due to gravity, which is handled by `BlockStateProperties#LEVEL_FLOWING` if the value is 8 and the fluid is not a source.

Implementation-wise, this means that the `$Flowing` and `$Source` subclasses are no longer needed, along with `FlowingFluid#getFlowing()` and `getSource()`. As each fluid now represents all possible states, tag comparison is no longer necessary and can be done directly by comparing the `Fluid`s via the `FluidState` passed around. Additionally, `Fluid`s now take in a `Fluid$Properties`, which is responsible for setting the associated bucket item, sound event on bucket pickup, the drip particle, its explosion resistance, whether the fluid randomly ticks, and the amount of drag it puts on entities moving through.

```java
// Assume we have some `ExampleFluid` extends `FlowingFluid` with the
// same constructor arguments.
public static final ExampleFluid EXAMPLE_FLUID = new ExampleFluid(
    // Construct the fluid properties
    Fluid.Properties.of()
        // The `ResourceKey` of the item representing the fluid in a bucket.
        .bucketItem(ItemIds.APPLE)
        // The `SoundEvent` to play when picked up by a bucket.
        // If not set, no sound is played.
        .soundEvent(SoundEvents.BUCKET_FILL)
        // The particle that drips through other blocks when this fluid is above.
        // If not set, there is no drip particle.
        .dripParticle(ParticleTypes.DRIPPING_WATER)
        // The explosion resistance of the fluid. When calculating whether the
        // associated block breaks, it will choose the maximum resistance between
        // the fluid and the block.
        // Defaults to 0f.
        .explosionResistance(100f)
        // When set, `Fluid#randomTick` is called based on the random tick speed
        // game rule.
        .randomTicking()
        // The amount of drag an entity feels when moving through the fluid. What
        // direction that drag is felt in depends on the movement implementation.
        // Defaults to 1f.
        .drag(0.8f)
);
```

## Debug Facts

The debug screen has been rewritten to better organize information into 'groups' with understandable contents, or 'facts'.

On its most basic level, `DebugScreenEntry`s add their elements to the `DebugScreenDisplayer` via `addToGroup` (replaces `addLine`) or `addFactToGroup`, which takes in a `DebugGroup` to display under, along with the information to display. For a given group, debug elements are displayed in the following order: any `DebugFact`s added via `addFactToGroup`, then lines added through `addToGroup`, and then finally the `DebugCustomRenderer` for non-text elements. As lines remain the same from previous versions, this will focus on `DebugFact`s and `DebugCustomRenderer`s.

`DebugFact` is essentially a `Component` wrapper, where a call to any of its methods to add data appends a sibling `Component`. There are many methods of doing so (e.g., `DebugFact#text`, `value`) which all delegate to `MutableComponent#append`. `DebugScreenDisplayer#addFactToGroup` constructs a fact by taking the `String` name of the fact, followed by a consumer to build the backing `Component`:

```java
// A debug entry example.
public class DebugExample implements DebugScreenEntry {

    @Override
    public void display(DebugScreenDisplayer displayer, @Nullable Level serverOrClientLevel, @Nullable LevelChunk clientChunk, @Nullable LevelChunk serverChunk) {
        // Adds a fact to display.
        // Will show up as 'Example Fact: Hello world!'.
        displayer.addFactToGroup(
            // The group to display the fact under in the debug menu.
            DebugGroups.MISC,
            // The name of the fact being displayed.
            // Will show up before the fact followed by a ':'.
            "Example Fact",
            // A builder to construct the debug fact component.
            fact -> fact.value("Hello world!")
        );
    }
}
```

`DebugCustomRenderer` displays last, used adding debug elements that cannot be represented as text. `extract` submits the elements to render to the screen. `height` and `width` meanwhile are used to properly align the element on screen (especially on the right side) such that no other debug information is drawn underneath, along with draw the background box for the group:


```java
// A debug entry example.
public class DebugExample implements DebugScreenEntry {

    @Override
    public void display(DebugScreenDisplayer displayer, @Nullable Level serverOrClientLevel, @Nullable LevelChunk clientChunk, @Nullable LevelChunk serverChunk) {
        // Adds a custom renderer to display items.
        displayer.addFactToGroup(
            // The group to display the fact under in the debug menu.
            DebugGroups.MISC,
            // The debug renderer to display in the group.
            new DebugCustomRenderer() {

                @Override
                public void extract(GuiGraphicsExtractor graphics, int left, int top, DebugColumn.Side side) {
                    // Submit the items to display.
                    // 'left' represents the starting X of the group.
                    // 'top' represents the starting Y of the element in the group.
                    graphics.item(new ItemStack(Items.DIRT), left, top);
                }

                @Override
                public int height() {
                    // The height of the element to render.
                    // This is technically oversimplified as it should be using the bounds
                    // initialized in `GuiItemRenderState`, which is not easily accessible.
                    return 16;
                }

                @Override
                public int width(int groupWidth) {
                    // The width of the element to render.
                    // This is technically oversimplified as it should be using the bounds
                    // initialized in `GuiItemRenderState`, which is not easily accessible.
                    return 16;
                }
            }
        );
    }
}
```

Custom `DebugGroup`s can be created quite easily via `DebugGroup$Builder`, calling either `titled` if the group should have a title header, or `titleless` if no header is given. Then, along with any configurations, the group is built by `build`, which can then be added to the `DebugScreenDisplayer`:


```java
// A debug group example.
// Note that the display order of debug groups is just whichever is encountered
// first in the debug entries.
public static final DebugGroup EXAMPLE_GROUP = DebugGroup.Builder.titled(
    // The name of the group.
    // Will be displayed left-aligned before any lines, facts, or renderers.
    "Example Group"
).withAccentColor(
    // The RGB color to act as a window sidebar spanning the group height.
    // Will be displayed on the left or right depending on which side its on.
    0xFF00FF
).withPreferredColumn(
    // The side the group will be displayed on, as long as the column is not full.
    // If the column is full, then the opposite column will be used first, then alternating
    // depending on fullness.
    DebugColumn.Side.RIGHT
).build();

// A debug entry example.
public class DebugExample implements DebugScreenEntry {

    @Override
    public void display(DebugScreenDisplayer displayer, @Nullable Level serverOrClientLevel, @Nullable LevelChunk clientChunk, @Nullable LevelChunk serverChunk) {
        displayer.addToGroup(
            // The group to display the fact under in the debug menu.
            DebugGroups.EXAMPLE_GROUP,
            "Example text"
        );
    }
}
```

## Placement Domains

`PlacementModifier`s must now specify the horizontal domain where a feature can be placed. This is done through `modifyXzDomain`, which takes in the previous domain (either modified by a different `PlacementModifier` or `0` for the original domain) and returns the new domain. Vanilla uses the domains as a validity check to make sure features can only start with `[-16, 31]` of the origin for each chunk.

```java
// An example `PlacementModifier`.
public record CenteredOffsetPlacement(IntProvider xzSize, int center) implements PlacementModifier {
    // The type used to register the placement.
    public static final MapCodec<CenteredOffsetPlacement> CODEC = RecordCodecBuilder.mapCodec(
        instance -> instance.group(
            IntProviders.codec(1, 16).fieldOf("xz_size").forGetter(CenteredOffsetPlacement::xzSize),
            ExtraCodecs.intRange(0, 15).fieldOf("center").forGetter(CenteredOffsetPlacement::center)
        ).apply(instance, CenteredOffsetPlacement::new)
    );

    @Override
    public MapCodec<CenteredOffsetPlacement> codec() {
        return CODEC;
    }

    @Override
    public void modify(PlacementContext context, RandomSource random, BlockPos origin, Consumer<BlockPos> output) {
        // The modifier offsets the placement by the provided XZ size, subtracting the center value.
        int x = this.xzSize.sample(random);
        int z = this.xzSize.sample(random);
        output.accept(origin.offset(x - this.center, 0, z - this.center));
    }

    @Override
    public InclusiveRange<Integer> modifyXzDomain(InclusiveRange<Integer> inputDomain) {
        // Must match the horizontal domain.
        return new InclusiveRange<>(
            // The minimum possible value that can be outputted relative to the origin.
            inputDomain.minInclusive() + this.xzSize.minInclusive() - this.center,
            // The maximum possible value that can be outputted relative to the origin.
            inputDomain.maxInclusive() + this.xzSize.maxInclusive() - this.center
        );
    }
}
```

## Not a `RuleTestType`, but a `MapCodec`

Rule test types have joined the unrolling club, as `RuleTestType` is removed and replaced with a `MapCodec<RuleTest>`. As such, `RuleTest#getType` is now `codec` to return the registered `MapCodec`. Additionally, `RuleTest` is now an interface from an abstract class, meaning that the `codec` method is now `public`. All other signatures remain the same.

```java
// A Rule test example.
public record RandomTest(float probability) implements RuleTest {
    // The map codec used as the registry object.
    public static final MapCodec<RandomTest> CODEC = Codec.FLOAT.fieldOf("probability")
        .xmap(RandomTest::new, RandomTest::probability);

    @Override
    public boolean test(BlockState state, BlockPos pos, RandomSource random) {
        return random.nextFloat() < this.probability;
    }

    // Replaces `getType`.
    @Override
    public MapCodec<RandomTest> codec() {
        // Return the registry object.
        return CODEC;
    }
}

// Register the map codec to the appropriate registry.
Registry.register(
    BuiltInRegistries.RULE_TEST_TYPE,
    Identifier.fromNamespaceAndPath("examplemod", "random"),
    RandomTest.CODEC
);
```

## Pathfinding Tags

Some `PathType`s now have an associated block tag `minecraft:pathfinding/*`, allowing entity navigation to treat the block as belonging to a specific group it can pathfind through or around. Most `PathType`s still remain explicitly checked for or computed.

## Biomes are Noise Biomes, but also Biomes

A large portion of the codebase around chunk generators resolving biomes has been further specified. Most of the 'biome resolver' classes and methods were renamed to 'noise biome resolvers' (e.g., `BiomeResolver` -> `NoiseBiomeResolver`). However, the original biome resolver methods and classes still remain, just under the connotation that the backing data used to resolve the biome may not specifically be directly from the generated noise.
