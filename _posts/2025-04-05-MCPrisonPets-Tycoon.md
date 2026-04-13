---
layout: post
author: qbb84
tags: ["Java", "Spigot", "Minecraft", "Minigames", "Game Design"]
---

*A concise dev log covering PrisonPets — a Spigot minigame combining a prison mining loop, tiered pet collection, player economy, and MMO-inspired raid mechanics driven by pet abilities.*

## The Concept

The core loop is simple: mine, collect pets, upgrade, raid. Each layer feeds the next — mining surfaces pets and resources, pets power raid encounters, raids yield rare rewards that feed back into the economy. The design is deliberately inspired by idle tycoon and collection mechanics, adapted for a Minecraft server environment where the interaction model is spatial and real-time rather than menu-driven.

The constraint I worked to: every system should be readable to a new player in under a minute. Minecraft players have limited tolerance for UI complexity.

## Pet Rarity System

Eight rarity tiers — `COMMON` through `PRISMATIC` — each with a weighted drop chance and a colour used consistently across all UI surfaces:

```java
public enum PetRarity {
    COMMON(ChatColor.GRAY, "Common", 50.0),
    UNCOMMON(ChatColor.GREEN, "Uncommon", 30.0),
    RARE(ChatColor.BLUE, "Rare", 15.0),
    EPIC(ChatColor.DARK_PURPLE, "Epic", 4.0),
    LEGENDARY(ChatColor.GOLD, "Legendary", 0.8),
    MYTHICAL(ChatColor.RED, "Mythical", 0.15),
    CELESTIAL(ChatColor.AQUA, "Celestial", 0.04),
    PRISMATIC(ChatColor.LIGHT_PURPLE, "Prismatic", 0.01);
    ...
}
```

The drop chances sum to 100.0. Rolling a pet calls `getRandomRarity()`, which accumulates probability mass across tiers until the random value is exceeded — standard weighted random selection.

The more interesting variant is `getRandomRarity(double luckMultiplier)`, which takes a luck value from the player's equipped pet stats and scales the probability distribution toward higher tiers:

```java
for (int i = 0; i < rarities.length; i++) {
    double tierBonus = 1.0 + (i * 0.2);
    adjustedChances[i] = rarities[i].getDropChance() * Math.pow(luckMultiplier, tierBonus);
    totalChance += adjustedChances[i];
}
```

Higher-tier rarities get a larger exponent applied to the luck multiplier — meaning luck has a compounding effect on rare drops rather than a flat shift. A lucky pet meaningfully changes your odds of finding a Prismatic without trivialising the rarity curve at the Common/Uncommon end.

## Pet Architecture

Each `Pet` instance holds its own stat map, ability list, level, XP, evolution ID, mining power, and arena bonus. Stats are initialised from the pet's type and rarity at construction, then scaled with level:

```java
public int getRequiredXPForNextLevel() {
    return (int) (100 * Math.pow(level, 1.5));
}
```

The polynomial XP curve means early levels are fast and later levels slow down meaningfully — the same pattern used in most MMO progression systems. Level is capped at 100.

Type determines the pet's role in the minigame loop:

- `ATTACK` — +2% arena bonus base, scales into raid damage
- `MINING` — +2 base mining power, surfaces more pets per session
- `SUPPORT` / `TANK` / `SPEED` — utility roles with distinct stat profiles

Each pet exposes `getEffectiveMiningPower()` and `getEffectiveArenaBonus()` — derived values that fold in the level bonus — so the rest of the codebase never has to re-derive these:

```java
public int getEffectiveMiningPower() {
    return (int) (miningPower * (1 + (level - 1) * 0.05));
}
```

Pets are serialised and deserialised via `saveToConfig` / `loadFromConfig` using Bukkit's `ConfigurationSection` API, keeping persistence straightforward without a database dependency at this scale.

## PetTemplate

`PetTemplate` decouples the definition of a pet species from the instance of a player's specific pet. Templates define the archetype — name, rarity, type, base stats, abilities, evolution chain — and `createPet()` stamps out a fresh `Pet` instance from the template. This is the standard Prototype pattern applied to game object instantiation.

## The Raid System

Raids are the endgame cooperative loop. An `ActiveRaid` holds the live state of an in-progress encounter:

```java
public class ActiveRaid {
    private final Set<UUID> participants = new HashSet<>();
    private final Map<UUID, Integer> participantDamage = new HashMap<>();
    private RaidState state = RaidState.WAITING;
    private RaidBoss boss;
    ...
}
```

`RaidState` progresses through `WAITING → READY → IN_PROGRESS → COMPLETED / FAILED`. The state machine is simple and explicit — no ambiguity about what a raid instance is doing at any point.

Damage tracking is per-participant, which enables a top damage leaderboard at completion:

```java
public List<UUID> getTopDamageDealers(int count) {
    return participantDamage.entrySet().stream()
        .sorted(Map.Entry.comparingByValue(Comparator.reverseOrder()))
        .limit(count)
        .map(Map.Entry::getKey)
        .collect(Collectors.toList());
}
```

This feeds the reward distribution — top contributors receive better loot tiers, incentivising active participation over passive presence.

`RaidBoss` carries its own rarity — a boss's rarity tier gates which pet eggs can drop as completion rewards, creating a direct link between the difficulty of the encounter and the quality of the loot pool.

Damage to the boss is clamped to current health so overkill is always handled gracefully:

```java
public int takeDamage(int damage) {
    int actualDamage = Math.min(currentHealth, damage);
    currentHealth -= actualDamage;
    return actualDamage;
}
```

## Mine Regions

Mines are defined as bounded world-space regions — `MineRegion` holds a min/max `Location` pair and exposes a `contains(Location)` check used to determine when a player is actively mining within a valid zone:

```java
return location.getX() >= min.getX() && location.getX() <= max.getX() &&
       location.getY() >= min.getY() && location.getY() <= max.getY() &&
       location.getZ() >= min.getZ() && location.getZ() <= max.getZ();
```

Regions auto-reset on a 5-minute timer (`needsReset()` checks `System.currentTimeMillis()` against `lastResetTime`), keeping mines populated without manual intervention.

## Trading

The `Trade` class supports bidirectional exchange of pets, items, coins, and gems — each tracked separately per player. Both parties must explicitly call `setAccepted(true)` before any transfer occurs. The `TradeManager` holds all active trades keyed by participant UUID, with a `TradeRequest` intermediary for the invitation flow before a trade session opens.

Pets are deep-copied via `Pet.copy()` before being placed into a trade — ensuring the original is never mutated until the trade settles. This prevents a race condition where a pet's state changes between being offered and the trade completing.

[View Repo](https://github.com/qbb84/PrisonPets)
