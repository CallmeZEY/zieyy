# Custom Commands Addon — Minecraft Bedrock

Addon Minecraft Bedrock yang menambahkan command custom untuk kebutuhan server survival/RPG: sistem ekonomi, home, market yang dilindungi penuh, jual-beli, rank player, sampai custom enchant di atas batas level vanilla.

Dibuat 100% pakai **Script API resmi Minecraft** (`@minecraft/server` & `@minecraft/server-ui`) — tanpa mod eksternal, tanpa modifikasi file game.

## ✨ Fitur

| Command | Akses | Fungsi |
|---|---|---|
| `/saldo` | Semua player | Lihat saldo dari scoreboard `money` |
| `/sethome` | Semua player | Tandai lokasi rumah |
| `/home` | Semua player | Teleport ke rumah (cooldown 5 detik setelah kena damage) |
| `/setmarket` | Operator saja | Tandai lokasi market |
| `/market` | Semua player | Teleport ke market |
| `/sell` | Semua player | Buka menu jual barang |
| `/rate <item> <harga>` | Operator saja | Atur harga jual sebuah item |
| `/crn <player> <rank>` | Operator saja | Ubah rank/tag nama seorang player |
| `/cenchant <player> <jenis> <level 1-10>` | Operator saja | Custom enchant ke item yang dipegang |
| `/cbook <player> <jenis> <level 1-10>` | Operator saja | Beri buku custom enchant langsung |
| `/online` | Semua player | Lihat daftar player online |

**Proteksi market** (radius 20 blok dari titik `/setmarket`): anti hancur manual, anti taruh block, anti ledakan, auto-bersihkan mob berbahaya, auto-padamkan api.

**Custom enchant** bisa naik sampai level 10 (angka romawi X), jauh di atas batas vanilla Minecraft (misal Sharpness biasanya maksimal V). 14 jenis enchant sudah punya efek bonus nyata: sharpness, smite, bane of arthropods, knockback, punch, power, protection, feather falling, thorns, efficiency, unbreaking, fortune, looting, respiration.

## 📦 Instalasi

1. Download file `.mcpack` dari [Releases](../../releases)
2. Buka file tersebut — otomatis ter-import ke Minecraft
3. Saat membuat/mengedit world: aktifkan pack ini di **Behavior Packs**
4. Di tab **Experiments**, aktifkan **"Beta APIs"** (wajib, addon tidak akan jalan tanpa ini)
5. Mainkan world-nya

**Requirement:** Minecraft Bedrock versi **1.21.100** ke atas.

## 🛠️ Development

Struktur project:
```
.
├── manifest.json          # Metadata & dependency behavior pack
├── pack_icon.png           # Icon pack
└── scripts/
    └── main.js             # Seluruh logika addon (single file)
```
JavaScriptcode
```
import {
  world,
  system,
  CommandPermissionLevel,
  CustomCommandStatus,
  CustomCommandParamType,
  Player,
  EquipmentSlot,
  EntityComponentTypes,
  ItemComponentTypes,
  ItemStack,
  EnchantmentType,
} from "@minecraft/server";
import { ActionFormData, ModalFormData } from "@minecraft/server-ui";

/* =========================================================
   PENGATURAN (silakan ubah sesuai kebutuhanmu)
   ========================================================= */

const OBJECTIVE_ID = "money";               // nama objective scoreboard saldo
const HOME_KEY = "home:location";           // dynamic property player
const MARKET_KEY = "market:location";       // dynamic property world
const SELL_RATES_KEY = "sell:rates";        // dynamic property world (JSON map itemId -> harga)
const RANK_KEY = "rank:tag";                // dynamic property player

const DAMAGE_COOLDOWN_SECONDS = 5;          // cooldown /home & /market setelah kena damage
const DAMAGE_COOLDOWN_TICKS = DAMAGE_COOLDOWN_SECONDS * 20;

const MARKET_PROTECT_RADIUS = 20;           // radius proteksi market (blok), tiap mata angin
const MARKET_MOB_SCAN_INTERVAL_TICKS = 100; // tiap 5 detik, bersihkan mob berbahaya
const MARKET_FIRE_SCAN_INTERVAL_TICKS = 100;// tiap 5 detik, padamkan api di area market
const MARKET_FIRE_SCAN_Y_RANGE = 10;        // jangkauan atas/bawah saat memadamkan api (performa)

const DEFAULT_RANK = "Member";

/* =========================================================
   UTIL UMUM
   ========================================================= */

function serializeLocation(location, dimension) {
  return JSON.stringify({ x: location.x, y: location.y, z: location.z, dimension: dimension.id });
}

function deserializeLocation(raw) {
  try {
    const data = JSON.parse(raw);
    if (typeof data.x !== "number" || typeof data.y !== "number" || typeof data.z !== "number" || typeof data.dimension !== "string") {
      return null;
    }
    return data;
  } catch (e) {
    return null;
  }
}

function formatCoords(location) {
  return `${Math.floor(location.x)}, ${Math.floor(location.y)}, ${Math.floor(location.z)}`;
}

function ensureObjective() {
  if (!world.scoreboard.getObjective(OBJECTIVE_ID)) {
    try {
      world.scoreboard.addObjective(OBJECTIVE_ID, "Saldo");
    } catch (e) {
      /* sudah ada, aman diabaikan */
    }
  }
}

function addMoney(player, amount) {
  ensureObjective();
  const objective = world.scoreboard.getObjective(OBJECTIVE_ID);
  const current = objective.getScore(player) ?? 0;
  objective.setScore(player, current + amount);
  return current + amount;
}

function prettifyItemName(itemId) {
  const short = itemId.replace("minecraft:", "");
  return short
    .split("_")
    .map((w) => w.charAt(0).toUpperCase() + w.slice(1))
    .join(" ");
}

/* =========================================================
   COOLDOWN DAMAGE (/home & /market)
   ========================================================= */

const lastDamageTick = new Map();

world.afterEvents.entityHurt.subscribe((event) => {
  const entity = event.hurtEntity;
  if (entity && entity.typeId === "minecraft:player") {
    lastDamageTick.set(entity.id, system.currentTick);
  }
});

function getRemainingCooldown(player) {
  const last = lastDamageTick.get(player.id);
  if (last === undefined) return 0;
  const elapsed = system.currentTick - last;
  if (elapsed < DAMAGE_COOLDOWN_TICKS) {
    return Math.ceil((DAMAGE_COOLDOWN_TICKS - elapsed) / 20);
  }
  return 0;
}

/* =========================================================
   SISTEM RANK / TAG NAMA
   ========================================================= */

function getRank(player) {
  const raw = player.getDynamicProperty(RANK_KEY);
  return typeof raw === "string" && raw.length > 0 ? raw : DEFAULT_RANK;
}

function applyRankNameTag(player) {
  const rank = getRank(player);
  player.nameTag = `§7[§b${rank}§7] §f${player.name}`;
}

world.afterEvents.playerSpawn.subscribe((event) => {
  const { player, initialSpawn } = event;
  if (initialSpawn) {
    if (player.getDynamicProperty(RANK_KEY) === undefined) {
      player.setDynamicProperty(RANK_KEY, DEFAULT_RANK);
    }
  }
  applyRankNameTag(player);
});

/* =========================================================
   PROTEKSI MARKET (anti hancur + bersih mob berbahaya)
   ========================================================= */

const HOSTILE_TYPE_IDS = new Set([
  "minecraft:zombie", "minecraft:husk", "minecraft:drowned", "minecraft:skeleton",
  "minecraft:stray", "minecraft:creeper", "minecraft:spider", "minecraft:cave_spider",
  "minecraft:enderman", "minecraft:witch", "minecraft:slime", "minecraft:magma_cube",
  "minecraft:phantom", "minecraft:pillager", "minecraft:vindicator", "minecraft:evoker",
  "minecraft:ravager", "minecraft:blaze", "minecraft:ghast", "minecraft:silverfish",
  "minecraft:guardian", "minecraft:elder_guardian", "minecraft:hoglin", "minecraft:zoglin",
  "minecraft:piglin_brute", "minecraft:vex", "minecraft:wither_skeleton", "minecraft:shulker",
]);

function getMarketData() {
  const raw = world.getDynamicProperty(MARKET_KEY);
  if (typeof raw !== "string") return null;
  return deserializeLocation(raw);
}

function isWithinMarketRadius(location, dimensionId, marketData) {
  if (!marketData || marketData.dimension !== dimensionId) return false;
  return (
    Math.abs(location.x - marketData.x) <= MARKET_PROTECT_RADIUS &&
    Math.abs(location.y - marketData.y) <= MARKET_PROTECT_RADIUS &&
    Math.abs(location.z - marketData.z) <= MARKET_PROTECT_RADIUS
  );
}

// Cegah player menghancurkan block di area market
world.beforeEvents.playerBreakBlock.subscribe((event) => {
  const marketData = getMarketData();
  if (!marketData) return;
  const block = event.block;
  if (isWithinMarketRadius(block.location, event.player.dimension.id, marketData)) {
    event.cancel = true;
    event.player.sendMessage("§c[Market] Area market dilindungi, tidak bisa dihancurkan.");
  }
});

// Cegah player menaruh block baru di area market
// Catatan: world.beforeEvents.playerPlaceBlock adalah API BETA dan tidak tersedia
// di channel stable, jadi kita pakai afterEvents (stable) lalu langsung membatalkan
// efeknya secara manual: block dikembalikan ke semula & item dikembalikan ke player.
world.afterEvents.playerPlaceBlock.subscribe((event) => {
  const marketData = getMarketData();
  if (!marketData) return;
  const { player, block } = event;
  if (!isWithinMarketRadius(block.location, player.dimension.id, marketData)) return;

  const placedTypeId = block.typeId;
  system.run(() => {
    try {
      block.setType("minecraft:air");
      const inventory = player.getComponent(EntityComponentTypes.Inventory)?.container;
      if (inventory) {
        inventory.addItem(new ItemStack(placedTypeId, 1));
      }
      player.sendMessage("§c[Market] Area market dilindungi, tidak bisa menaruh block di sini.");
    } catch (e) {
      /* aman diabaikan */
    }
  });
});

// Cegah ledakan menghancurkan block di area market
world.beforeEvents.explosion.subscribe((event) => {
  const marketData = getMarketData();
  if (!marketData) return;
  const dimensionId = event.dimension.id;
  const impacted = event.getImpactedBlocks();
  const filtered = impacted.filter((block) => !isWithinMarketRadius(block.location, dimensionId, marketData));
  if (filtered.length !== impacted.length) {
    event.setImpactedBlocks(filtered);
  }
});

// Bersihkan mob berbahaya di sekitar market secara berkala
system.runInterval(() => {
  const marketData = getMarketData();
  if (!marketData) return;
  try {
    const dimension = world.getDimension(marketData.dimension);
    const center = { x: marketData.x, y: marketData.y, z: marketData.z };
    const entities = dimension.getEntities({ location: center, maxDistance: MARKET_PROTECT_RADIUS });
    for (const entity of entities) {
      if (HOSTILE_TYPE_IDS.has(entity.typeId)) {
        entity.remove();
      }
    }
  } catch (e) {
    /* dimension mungkin belum dimuat, aman diabaikan */
  }
}, MARKET_MOB_SCAN_INTERVAL_TICKS);

// Padamkan api di area market secara berkala (bukan real-time, demi performa)
system.runInterval(() => {
  const marketData = getMarketData();
  if (!marketData) return;
  try {
    const dimension = world.getDimension(marketData.dimension);
    const cx = Math.floor(marketData.x);
    const cy = Math.floor(marketData.y);
    const cz = Math.floor(marketData.z);
    const yMin = cy - MARKET_FIRE_SCAN_Y_RANGE;
    const yMax = cy + MARKET_FIRE_SCAN_Y_RANGE;
    for (let x = cx - MARKET_PROTECT_RADIUS; x <= cx + MARKET_PROTECT_RADIUS; x++) {
      for (let z = cz - MARKET_PROTECT_RADIUS; z <= cz + MARKET_PROTECT_RADIUS; z++) {
        for (let y = yMin; y <= yMax; y++) {
          const block = dimension.getBlock({ x, y, z });
          if (block && (block.typeId === "minecraft:fire" || block.typeId === "minecraft:soul_fire")) {
            block.setType("minecraft:air");
          }
        }
      }
    }
  } catch (e) {
    /* aman diabaikan */
  }
}, MARKET_FIRE_SCAN_INTERVAL_TICKS);

/* =========================================================
   CUSTOM ENCHANT (level 1-10, level >5 = bonus dari script)
   ========================================================= */

const ENCHANTS = {
  sharpness: { id: "minecraft:sharpness", max: 5, label: "Sharpness" },
  smite: { id: "minecraft:smite", max: 5, label: "Smite" },
  bane_of_arthropods: { id: "minecraft:bane_of_arthropods", max: 5, label: "Bane of Arthropods" },
  knockback: { id: "minecraft:knockback", max: 2, label: "Knockback" },
  protection: { id: "minecraft:protection", max: 4, label: "Protection" },
  efficiency: { id: "minecraft:efficiency", max: 5, label: "Efficiency" },
  unbreaking: { id: "minecraft:unbreaking", max: 3, label: "Unbreaking" },
  fortune: { id: "minecraft:fortune", max: 3, label: "Fortune" },
  looting: { id: "minecraft:looting", max: 3, label: "Looting" },
  power: { id: "minecraft:power", max: 5, label: "Power" },
  punch: { id: "minecraft:punch", max: 2, label: "Punch" },
  respiration: { id: "minecraft:respiration", max: 3, label: "Respiration" },
  thorns: { id: "minecraft:thorns", max: 3, label: "Thorns" },
  feather_falling: { id: "minecraft:feather_falling", max: 4, label: "Feather Falling" },
};

const ROMAN = ["", "I", "II", "III", "IV", "V", "VI", "VII", "VIII", "IX", "X"];
function toRoman(n) {
  return ROMAN[n] ?? String(n);
}

// Sekarang SEMUA jenis enchant di daftar ini sudah punya efek bonus nyata untuk level di atas batas vanilla.
const ENCHANT_BONUS_SUPPORTED = new Set([
  "sharpness", "knockback", "protection", "feather_falling", "thorns", "power", "punch",
  "smite", "bane_of_arthropods", "efficiency", "unbreaking", "fortune", "looting", "respiration",
]);

const UNDEAD_TYPE_IDS = new Set([
  "minecraft:zombie", "minecraft:zombie_villager", "minecraft:husk", "minecraft:drowned",
  "minecraft:skeleton", "minecraft:stray", "minecraft:wither_skeleton", "minecraft:phantom",
  "minecraft:zoglin", "minecraft:wither",
]);
const ARTHROPOD_TYPE_IDS = new Set([
  "minecraft:spider", "minecraft:cave_spider", "minecraft:silverfish", "minecraft:endermite",
]);

const CUSTOM_ENCHANT_DP = "custom_enchants"; // dynamic property di item, isi JSON {key: level}

function readCustomEnchants(itemStack) {
  try {
    const raw = itemStack.getDynamicProperty(CUSTOM_ENCHANT_DP);
    if (typeof raw !== "string") return {};
    return JSON.parse(raw);
  } catch (e) {
    return {};
  }
}

// Jumlahkan bonus level (di atas batas vanilla) sebuah jenis enchant dari semua armor yang dipakai player
function sumArmorEnchantExtra(player, enchantKey) {
  const def = ENCHANTS[enchantKey];
  if (!def) return 0;
  const equippable = player.getComponent(EntityComponentTypes.Equippable);
  if (!equippable) return 0;
  const slots = [EquipmentSlot.Head, EquipmentSlot.Chest, EquipmentSlot.Legs, EquipmentSlot.Feet];
  let total = 0;
  for (const slot of slots) {
    try {
      const item = equippable.getEquipment(slot);
      if (!item) continue;
      const map = readCustomEnchants(item);
      const level = map[enchantKey];
      if (level && level > def.max) total += level - def.max;
    } catch (e) {
      /* aman diabaikan */
    }
  }
  return total;
}

function applyCustomEnchant(itemStack, enchantKey, level) {
  const def = ENCHANTS[enchantKey];
  if (!def) return { ok: false, message: "Jenis enchant tidak dikenal." };
  if (level < 1 || level > 10) return { ok: false, message: "Level harus 1-10." };

  const enchantable = itemStack.getComponent(ItemComponentTypes.Enchantable);
  if (!enchantable) return { ok: false, message: "Item ini tidak bisa di-enchant." };

  const engineLevel = Math.min(level, def.max);
  const extra = level - def.max;

  try {
    if (enchantable.hasEnchantment(def.id)) {
      enchantable.removeEnchantment(def.id);
    }
    enchantable.addEnchantment({ type: new EnchantmentType(def.id), level: engineLevel });
  } catch (e) {
    return { ok: false, message: "Gagal menerapkan enchant: " + e };
  }

  const map = readCustomEnchants(itemStack);
  if (extra > 0) {
    map[enchantKey] = level;
  } else {
    delete map[enchantKey];
  }
  itemStack.setDynamicProperty(CUSTOM_ENCHANT_DP, JSON.stringify(map));

  // Rebuild lore berdasarkan semua custom enchant yang tersimpan di item ini
  const lore = [];
  for (const key of Object.keys(map)) {
    const d = ENCHANTS[key];
    if (d) lore.push(`§7${d.label} ${toRoman(map[key])}§r`);
  }
  itemStack.setLore(lore);

  return { ok: true, engineLevel, extra };
}

// Bonus damage & knockback untuk level di atas batas vanilla (6-10)
const EXTRA_DAMAGE_PER_LEVEL = 1; // tiap level custom di atas batas vanilla = +1 attack damage
const EXTRA_KNOCKBACK_PER_LEVEL = 0.35;

world.afterEvents.entityHitEntity.subscribe((event) => {
  const attacker = event.damagingEntity;
  const target = event.hitEntity; // PENTING: properti aslinya "hitEntity", bukan "hurtEntity"
  if (!attacker || !(attacker instanceof Player) || !target) return;

  system.run(() => {
    try {
      const equippable = attacker.getComponent(EntityComponentTypes.Equippable);
      const item = equippable?.getEquipment(EquipmentSlot.Mainhand);
      if (!item) return;
      const map = readCustomEnchants(item);

      if (map.sharpness && map.sharpness > ENCHANTS.sharpness.max) {
        const extraLevels = map.sharpness - ENCHANTS.sharpness.max;
        const bonusDamage = extraLevels * EXTRA_DAMAGE_PER_LEVEL;
        target.applyDamage(bonusDamage, { damagingEntity: attacker, cause: "entityAttack" });
      }

      if (map.knockback && map.knockback > ENCHANTS.knockback.max) {
        const extraLevels = map.knockback - ENCHANTS.knockback.max;
        const dx = target.location.x - attacker.location.x;
        const dz = target.location.z - attacker.location.z;
        const length = Math.sqrt(dx * dx + dz * dz) || 1;
        target.applyKnockback(
          { x: dx / length, z: dz / length },
          EXTRA_KNOCKBACK_PER_LEVEL * extraLevels
        );
      }

      if (map.smite && map.smite > ENCHANTS.smite.max && UNDEAD_TYPE_IDS.has(target.typeId)) {
        const extraLevels = map.smite - ENCHANTS.smite.max;
        target.applyDamage(extraLevels * EXTRA_DAMAGE_PER_LEVEL, { damagingEntity: attacker, cause: "entityAttack" });
      }

      if (map.bane_of_arthropods && map.bane_of_arthropods > ENCHANTS.bane_of_arthropods.max && ARTHROPOD_TYPE_IDS.has(target.typeId)) {
        const extraLevels = map.bane_of_arthropods - ENCHANTS.bane_of_arthropods.max;
        target.applyDamage(extraLevels * EXTRA_DAMAGE_PER_LEVEL, { damagingEntity: attacker, cause: "entityAttack" });
      }
    } catch (e) {
      /* item mungkin tidak valid lagi, aman diabaikan */
    }
  });
});

// Bonus armor: Protection (semua damage kecuali fall), Feather Falling (khusus fall),
// dan Thorns (reflect damage ke penyerang) untuk level di atas batas vanilla.
world.afterEvents.entityHurt.subscribe((event) => {
  const { hurtEntity, damageSource, damage } = event;
  if (!(hurtEntity instanceof Player)) return;

  system.run(() => {
    try {
      const isFall = damageSource?.cause === "fall";
      const reduceKey = isFall ? "feather_falling" : "protection";
      const reduceExtra = sumArmorEnchantExtra(hurtEntity, reduceKey);

      if (reduceExtra > 0 && damage > 0) {
        const reduction = Math.min(0.8, reduceExtra * 0.04); // mirip skala protection vanilla (~4%/level)
        const healBack = damage * reduction;
        const health = hurtEntity.getComponent(EntityComponentTypes.Health);
        if (health && healBack > 0) {
          const newValue = Math.min(health.currentValue + healBack, health.effectiveMax);
          health.setCurrentValue(newValue);
        }
      }

      const thornsExtra = sumArmorEnchantExtra(hurtEntity, "thorns");
      if (thornsExtra > 0 && damageSource?.damagingEntity) {
        damageSource.damagingEntity.applyDamage(thornsExtra * 1.0, {
          damagingEntity: hurtEntity,
          cause: "entityAttack",
        });
      }
    } catch (e) {
      /* aman diabaikan */
    }
  });
});

// Bonus panah: Power (damage tambahan) & Punch (knockback tambahan) untuk level di atas batas vanilla.
world.afterEvents.projectileHitEntity.subscribe((event) => {
  const shooter = event.source;
  if (!(shooter instanceof Player)) return;
  const hitInfo = event.getEntityHit();
  const target = hitInfo?.entity;
  if (!target) return;

  system.run(() => {
    try {
      const equippable = shooter.getComponent(EntityComponentTypes.Equippable);
      const bow = equippable?.getEquipment(EquipmentSlot.Mainhand);
      if (!bow) return;
      const map = readCustomEnchants(bow);

      if (map.power && map.power > ENCHANTS.power.max) {
        const extraLevels = map.power - ENCHANTS.power.max;
        target.applyDamage(extraLevels * 1.0, { damagingEntity: shooter, cause: "entityAttack" });
      }

      if (map.punch && map.punch > ENCHANTS.punch.max) {
        const extraLevels = map.punch - ENCHANTS.punch.max;
        const dx = target.location.x - shooter.location.x;
        const dz = target.location.z - shooter.location.z;
        const length = Math.sqrt(dx * dx + dz * dz) || 1;
        target.applyKnockback({ x: dx / length, z: dz / length }, 0.3 * extraLevels);
      }
    } catch (e) {
      /* aman diabaikan */
    }
  });
});

// Helper umum: kemungkinan sebuah drop item digandakan (dipakai Fortune & Looting)
function tryDuplicateNearbyDrops(dimension, location, extraLevels) {
  try {
    const chance = Math.min(0.9, extraLevels * 0.15);
    const drops = dimension.getEntities({ type: "item", location, maxDistance: 3 });
    for (const dropEntity of drops) {
      if (Math.random() < chance) {
        const itemComp = dropEntity.getComponent(EntityComponentTypes.Item);
        if (itemComp?.itemStack) {
          dimension.spawnItem(itemComp.itemStack.clone(), dropEntity.location);
        }
      }
    }
  } catch (e) {
    /* aman diabaikan */
  }
}

// Helper Unbreaking: beri kesempatan tambahan agar durability tidak berkurang
function maybeApplyUnbreaking(item) {
  try {
    const durability = item.getComponent(ItemComponentTypes.Durability);
    if (!durability) return;
    const map = readCustomEnchants(item);
    const extra = map.unbreaking ? map.unbreaking - ENCHANTS.unbreaking.max : 0;
    if (extra <= 0) return;
    const lastRaw = item.getDynamicProperty("ub_last_damage");
    const last = typeof lastRaw === "number" ? lastRaw : durability.damage;
    if (durability.damage > last) {
      const chance = Math.min(0.9, extra * 0.15);
      if (Math.random() < chance) {
        durability.damage = last; // batalkan kenaikan damage kali ini
      }
    }
    item.setDynamicProperty("ub_last_damage", durability.damage);
  } catch (e) {
    /* aman diabaikan */
  }
}

// Unbreaking saat menyerang (senjata) & Looting saat membunuh mob
world.afterEvents.entityDie.subscribe((event) => {
  const killer = event.damageSource?.damagingEntity;
  if (!(killer instanceof Player)) return;

  let deathLocation, deathDimension;
  try {
    deathLocation = event.deadEntity.location;
    deathDimension = event.deadEntity.dimension;
  } catch (e) {
    return;
  }

  system.run(() => {
    try {
      const equippable = killer.getComponent(EntityComponentTypes.Equippable);
      const weapon = equippable?.getEquipment(EquipmentSlot.Mainhand);
      if (!weapon) return;
      const map = readCustomEnchants(weapon);

      maybeApplyUnbreaking(weapon);
      equippable.setEquipment(EquipmentSlot.Mainhand, weapon);

      const lootingExtra = map.looting ? map.looting - ENCHANTS.looting.max : 0;
      if (lootingExtra > 0) {
        system.runTimeout(() => tryDuplicateNearbyDrops(deathDimension, deathLocation, lootingExtra), 4);
      }
    } catch (e) {
      /* aman diabaikan */
    }
  });
});

// Unbreaking saat menambang & Fortune saat memecah block
world.afterEvents.playerBreakBlock.subscribe((event) => {
  const { player, block, itemStackAfterBreak } = event;
  if (!itemStackAfterBreak) return;

  system.run(() => {
    try {
      const map = readCustomEnchants(itemStackAfterBreak);
      const equippable = player.getComponent(EntityComponentTypes.Equippable);

      maybeApplyUnbreaking(itemStackAfterBreak);
      if (equippable) equippable.setEquipment(EquipmentSlot.Mainhand, itemStackAfterBreak);

      const fortuneExtra = map.fortune ? map.fortune - ENCHANTS.fortune.max : 0;
      if (fortuneExtra > 0) {
        const dimension = player.dimension;
        const location = block.location;
        system.runTimeout(() => tryDuplicateNearbyDrops(dimension, location, fortuneExtra), 4);
      }
    } catch (e) {
      /* aman diabaikan */
    }
  });
});

// Efficiency (Haste sementara saat memegang tool) & Respiration (anti tenggelam saat di air)
system.runInterval(() => {
  for (const player of world.getAllPlayers()) {
    try {
      const equippable = player.getComponent(EntityComponentTypes.Equippable);
      if (!equippable) continue;

      const tool = equippable.getEquipment(EquipmentSlot.Mainhand);
      if (tool) {
        const toolMap = readCustomEnchants(tool);
        const effExtra = toolMap.efficiency ? toolMap.efficiency - ENCHANTS.efficiency.max : 0;
        if (effExtra > 0) {
          const amplifier = Math.min(4, effExtra - 1);
          player.addEffect("haste", 60, { amplifier: Math.max(0, amplifier), showParticles: false });
        }
      }

      if (player.isInWater) {
        const helmet = equippable.getEquipment(EquipmentSlot.Head);
        if (helmet) {
          const helmMap = readCustomEnchants(helmet);
          const respExtra = helmMap.respiration ? helmMap.respiration - ENCHANTS.respiration.max : 0;
          if (respExtra > 0) {
            player.addEffect("water_breathing", 60, { amplifier: 0, showParticles: false });
          }
        }
      }
    } catch (e) {
      /* aman diabaikan */
    }
  }
}, 20); // tiap 1 detik

/* =========================================================
   /sell dan /rate
   ========================================================= */

function getSellRates() {
  const raw = world.getDynamicProperty(SELL_RATES_KEY);
  if (typeof raw !== "string") return {};
  try {
    return JSON.parse(raw);
  } catch (e) {
    return {};
  }
}

function setSellRate(itemId, price) {
  const rates = getSellRates();
  if (price <= 0) {
    delete rates[itemId];
  } else {
    rates[itemId] = price;
  }
  world.setDynamicProperty(SELL_RATES_KEY, JSON.stringify(rates));
}

function countItemInInventory(container, itemId) {
  let total = 0;
  for (let i = 0; i < container.size; i++) {
    const stack = container.getItem(i);
    if (stack && stack.typeId === itemId) total += stack.amount;
  }
  return total;
}

function removeItemFromInventory(container, itemId, qty) {
  let remaining = qty;
  for (let i = 0; i < container.size && remaining > 0; i++) {
    const stack = container.getItem(i);
    if (stack && stack.typeId === itemId) {
      if (stack.amount <= remaining) {
        remaining -= stack.amount;
        container.setItem(i, undefined);
      } else {
        stack.amount -= remaining;
        container.setItem(i, stack);
        remaining = 0;
      }
    }
  }
  return qty - remaining;
}

function openSellMenu(player) {
  const rates = getSellRates();
  const itemIds = Object.keys(rates);
  const inventory = player.getComponent(EntityComponentTypes.Inventory).container;

  const sellable = itemIds
    .map((id) => ({ id, price: rates[id], count: countItemInInventory(inventory, id) }))
    .filter((entry) => entry.count > 0);

  if (sellable.length === 0) {
    player.sendMessage("§c[Sell] Kamu tidak punya barang yang bisa dijual di sini.");
    return;
  }

  const form = new ActionFormData()
    .title("Sell - Pasar")
    .body("Pilih barang yang ingin kamu jual:");
  for (const entry of sellable) {
    form.button(`${prettifyItemName(entry.id)}\n§eRp${entry.price}/item §7(Punya: ${entry.count})`);
  }

  form.show(player).then((response) => {
    if (response.canceled || response.selection === undefined) return;
    const chosen = sellable[response.selection];
    openSellQuantityMenu(player, chosen);
  });
}

function openSellQuantityMenu(player, entry) {
  const modal = new ModalFormData()
    .title(`Jual ${prettifyItemName(entry.id)}`)
    .slider("Jumlah yang dijual", 1, entry.count, { valueStep: 1, defaultValue: entry.count });

  modal.show(player).then((response) => {
    if (response.canceled || !response.formValues) return;
    const qty = Math.floor(response.formValues[0]);
    const inventory = player.getComponent(EntityComponentTypes.Inventory).container;
    const removed = removeItemFromInventory(inventory, entry.id, qty);
    if (removed <= 0) {
      player.sendMessage("§c[Sell] Gagal menjual barang.");
      return;
    }
    const total = removed * entry.price;
    const newBalance = addMoney(player, total);
    player.sendMessage(`§a[Sell] §fBerhasil menjual §e${removed}x ${prettifyItemName(entry.id)}§f seharga §eRp${total}§f. Saldo sekarang: §e${newBalance}`);
  });
}

/* =========================================================
   PENDAFTARAN CUSTOM COMMAND
   ========================================================= */

system.beforeEvents.startup.subscribe(({ customCommandRegistry }) => {
  customCommandRegistry.registerEnum("cmd:enchantType", Object.keys(ENCHANTS));

  /* ---------- /saldo ---------- */
  customCommandRegistry.registerCommand(
    { name: "cmd:saldo", description: "Menampilkan jumlah saldo/uang kamu", permissionLevel: CommandPermissionLevel.Any, cheatsRequired: false },
    (origin) => {
      const player = origin.sourceEntity;
      if (!(player instanceof Player)) return { status: CustomCommandStatus.Failure, message: "Command ini hanya bisa dijalankan oleh player." };
      system.run(() => {
        ensureObjective();
        const objective = world.scoreboard.getObjective(OBJECTIVE_ID);
        let score = 0;
        try { score = objective.getScore(player) ?? 0; } catch (e) { score = 0; }
        player.sendMessage(`§a[Saldo] §fUang kamu saat ini: §e${score}`);
      });
      return { status: CustomCommandStatus.Success };
    }
  );

  /* ---------- /sethome ---------- */
  customCommandRegistry.registerCommand(
    { name: "cmd:sethome", description: "Menandai lokasi rumah kamu di posisi saat ini", permissionLevel: CommandPermissionLevel.Any, cheatsRequired: false },
    (origin) => {
      const player = origin.sourceEntity;
      if (!(player instanceof Player)) return { status: CustomCommandStatus.Failure, message: "Command ini hanya bisa dijalankan oleh player." };
      system.run(() => {
        const location = player.location;
        const dimension = player.dimension;
        player.setDynamicProperty(HOME_KEY, serializeLocation(location, dimension));
        player.sendMessage(`§a[Home] §fLokasi rumah berhasil ditandai di §e${formatCoords(location)}`);
      });
      return { status: CustomCommandStatus.Success };
    }
  );

  /* ---------- /home ---------- */
  customCommandRegistry.registerCommand(
    { name: "cmd:home", description: "Teleport ke lokasi rumah yang sudah ditandai", permissionLevel: CommandPermissionLevel.Any, cheatsRequired: false },
    (origin) => {
      const player = origin.sourceEntity;
      if (!(player instanceof Player)) return { status: CustomCommandStatus.Failure, message: "Command ini hanya bisa dijalankan oleh player." };
      system.run(() => {
        const cooldown = getRemainingCooldown(player);
        if (cooldown > 0) {
          player.sendMessage(`§c[Home] Kamu baru saja terkena damage! Tunggu §e${cooldown}§c detik lagi.`);
          return;
        }
        const raw = player.getDynamicProperty(HOME_KEY);
        if (typeof raw !== "string") {
          player.sendMessage("§c[Home] Kamu belum menandai rumah. Gunakan §e/sethome §cterlebih dahulu.");
          return;
        }
        const data = deserializeLocation(raw);
        if (!data) {
          player.sendMessage("§c[Home] Data lokasi rumah rusak, silakan set ulang dengan /sethome.");
          return;
        }
        const dimension = world.getDimension(data.dimension);
        player.teleport({ x: data.x, y: data.y, z: data.z }, { dimension });
        player.sendMessage("§a[Home] §fBerhasil teleport ke rumah.");
      });
      return { status: CustomCommandStatus.Success };
    }
  );

  /* ---------- /setmarket ---------- */
  customCommandRegistry.registerCommand(
    { name: "cmd:setmarket", description: "(Operator) Menandai lokasi market di posisi saat ini", permissionLevel: CommandPermissionLevel.GameDirectors, cheatsRequired: false },
    (origin) => {
      const player = origin.sourceEntity;
      if (!(player instanceof Player)) return { status: CustomCommandStatus.Failure, message: "Command ini hanya bisa dijalankan oleh player." };
      system.run(() => {
        const location = player.location;
        const dimension = player.dimension;
        world.setDynamicProperty(MARKET_KEY, serializeLocation(location, dimension));
        world.sendMessage(`§b[Market] §fLokasi market telah diperbarui oleh operator §e${player.name}§f di §e${formatCoords(location)}§f (radius proteksi ${MARKET_PROTECT_RADIUS} blok)`);
      });
      return { status: CustomCommandStatus.Success };
    }
  );

  /* ---------- /market ---------- */
  customCommandRegistry.registerCommand(
    { name: "cmd:market", description: "Teleport ke lokasi market yang sudah ditandai operator", permissionLevel: CommandPermissionLevel.Any, cheatsRequired: false },
    (origin) => {
      const player = origin.sourceEntity;
      if (!(player instanceof Player)) return { status: CustomCommandStatus.Failure, message: "Command ini hanya bisa dijalankan oleh player." };
      system.run(() => {
        const cooldown = getRemainingCooldown(player);
        if (cooldown > 0) {
          player.sendMessage(`§c[Market] Kamu baru saja terkena damage! Tunggu §e${cooldown}§c detik lagi.`);
          return;
        }
        const raw = world.getDynamicProperty(MARKET_KEY);
        if (typeof raw !== "string") {
          player.sendMessage("§c[Market] Lokasi market belum ditandai oleh operator.");
          return;
        }
        const data = deserializeLocation(raw);
        if (!data) {
          player.sendMessage("§c[Market] Data lokasi market rusak, minta operator set ulang dengan /setmarket.");
          return;
        }
        const dimension = world.getDimension(data.dimension);
        player.teleport({ x: data.x, y: data.y, z: data.z }, { dimension });
        player.sendMessage("§a[Market] §fBerhasil teleport ke market.");
      });
      return { status: CustomCommandStatus.Success };
    }
  );

  /* ---------- /sell ---------- */
  customCommandRegistry.registerCommand(
    { name: "cmd:sell", description: "Buka menu untuk menjual barang", permissionLevel: CommandPermissionLevel.Any, cheatsRequired: false },
    (origin) => {
      const player = origin.sourceEntity;
      if (!(player instanceof Player)) return { status: CustomCommandStatus.Failure, message: "Command ini hanya bisa dijalankan oleh player." };
      system.run(() => openSellMenu(player));
      return { status: CustomCommandStatus.Success };
    }
  );

  /* ---------- /rate ---------- */
  customCommandRegistry.registerCommand(
    {
      name: "cmd:rate",
      description: "(Operator) Mengatur harga jual sebuah item (harga 0 = hapus dari daftar jual)",
      permissionLevel: CommandPermissionLevel.GameDirectors,
      cheatsRequired: false,
      mandatoryParameters: [
        { name: "item", type: CustomCommandParamType.ItemType },
        { name: "harga", type: CustomCommandParamType.Integer },
      ],
    },
    (origin, itemType, harga) => {
      const player = origin.sourceEntity;
      if (!(player instanceof Player)) return { status: CustomCommandStatus.Failure, message: "Command ini hanya bisa dijalankan oleh player." };
      system.run(() => {
        setSellRate(itemType.id, harga);
        if (harga <= 0) {
          player.sendMessage(`§a[Rate] §f${prettifyItemName(itemType.id)} dihapus dari daftar jual.`);
        } else {
          player.sendMessage(`§a[Rate] §fHarga jual §e${prettifyItemName(itemType.id)}§f diatur menjadi §eRp${harga}§f/item.`);
        }
      });
      return { status: CustomCommandStatus.Success };
    }
  );

  /* ---------- /crn (custom rank) ---------- */
  customCommandRegistry.registerCommand(
    {
      name: "cmd:crn",
      description: "(Operator) Mengubah rank/tag seorang player",
      permissionLevel: CommandPermissionLevel.GameDirectors,
      cheatsRequired: false,
      mandatoryParameters: [
        { name: "target", type: CustomCommandParamType.PlayerSelector },
        { name: "rank", type: CustomCommandParamType.String },
      ],
    },
    (origin, targets, rank) => {
      const player = origin.sourceEntity;
      if (!(player instanceof Player)) return { status: CustomCommandStatus.Failure, message: "Command ini hanya bisa dijalankan oleh player." };
      const targetList = Array.isArray(targets) ? targets : [targets];
      if (targetList.length === 0) {
        return { status: CustomCommandStatus.Failure, message: "Player tidak ditemukan." };
      }
      system.run(() => {
        for (const target of targetList) {
          const cleanRank = rank.slice(0, 16);
          target.setDynamicProperty(RANK_KEY, cleanRank);
          applyRankNameTag(target);
          target.sendMessage(`§a[Rank] §fRank kamu diubah menjadi §b${cleanRank}§f oleh operator §e${player.name}`);
        }
        player.sendMessage(`§a[Rank] §fBerhasil mengubah rank ${targetList.length} player.`);
      });
      return { status: CustomCommandStatus.Success };
    }
  );

  /* ---------- /cenchant (custom enchant) ----------
     Catatan: sengaja TIDAK dinamai "enchant" karena Minecraft punya command
     bawaan bernama /enchant yang akan selalu menang jika terjadi bentrok nama.
  */
  customCommandRegistry.registerCommand(
    {
      name: "cmd:cenchant",
      description: "(Operator) Memberi custom enchant (level 1-10) ke item yang sedang dipegang target",
      permissionLevel: CommandPermissionLevel.GameDirectors,
      cheatsRequired: false,
      mandatoryParameters: [
        { name: "target", type: CustomCommandParamType.PlayerSelector },
        { name: "cmd:enchantType", type: CustomCommandParamType.Enum },
        { name: "level", type: CustomCommandParamType.Integer },
      ],
    },
    (origin, targets, enchantKey, level) => {
      const player = origin.sourceEntity;
      if (!(player instanceof Player)) return { status: CustomCommandStatus.Failure, message: "Command ini hanya bisa dijalankan oleh player." };
      const targetList = Array.isArray(targets) ? targets : [targets];
      if (targetList.length === 0) {
        return { status: CustomCommandStatus.Failure, message: "Player tidak ditemukan." };
      }
      system.run(() => {
        for (const target of targetList) {
          const equippable = target.getComponent(EntityComponentTypes.Equippable);
          const item = equippable?.getEquipment(EquipmentSlot.Mainhand);
          if (!item) {
            player.sendMessage(`§c[Enchant] ${target.name} tidak sedang memegang item apapun.`);
            continue;
          }
          const result = applyCustomEnchant(item, enchantKey, level);
          if (!result.ok) {
            player.sendMessage(`§c[Enchant] Gagal: ${result.message}`);
            continue;
          }
          equippable.setEquipment(EquipmentSlot.Mainhand, item);
          let noteExtra = "";
          if (result.extra > 0) {
            const supported = ENCHANT_BONUS_SUPPORTED.has(enchantKey);
            noteExtra = supported
              ? ` §7(level asli engine: ${result.engineLevel}, bonus efek script: +${result.extra})`
              : ` §7(level asli engine: ${result.engineLevel}, +${result.extra} baru tampil di lore, belum ada bonus efek)`;
          }
          player.sendMessage(`§a[Enchant] §f${ENCHANTS[enchantKey].label} ${toRoman(level)} diterapkan ke item ${target.name}.${noteExtra}`);
        }
      });
      return { status: CustomCommandStatus.Success };
    }
  );

  /* ---------- /cbook (buku custom enchant) ---------- */
  customCommandRegistry.registerCommand(
    {
      name: "cmd:cbook",
      description: "(Operator) Memberi buku enchanted book berisi custom enchant ke target",
      permissionLevel: CommandPermissionLevel.GameDirectors,
      cheatsRequired: false,
      mandatoryParameters: [
        { name: "target", type: CustomCommandParamType.PlayerSelector },
        { name: "cmd:enchantType", type: CustomCommandParamType.Enum },
        { name: "level", type: CustomCommandParamType.Integer },
      ],
    },
    (origin, targets, enchantKey, level) => {
      const player = origin.sourceEntity;
      if (!(player instanceof Player)) return { status: CustomCommandStatus.Failure, message: "Command ini hanya bisa dijalankan oleh player." };
      const targetList = Array.isArray(targets) ? targets : [targets];
      if (targetList.length === 0) {
        return { status: CustomCommandStatus.Failure, message: "Player tidak ditemukan." };
      }
      system.run(() => {
        for (const target of targetList) {
          const book = new ItemStack("minecraft:enchanted_book", 1);
          const result = applyCustomEnchant(book, enchantKey, level);
          if (!result.ok) {
            player.sendMessage(`§c[Enchant] Gagal membuat buku: ${result.message}`);
            continue;
          }
          const inventory = target.getComponent(EntityComponentTypes.Inventory)?.container;
          if (!inventory) {
            player.sendMessage(`§c[Enchant] Tidak bisa mengakses inventory ${target.name}.`);
            continue;
          }
          inventory.addItem(book);
          const noteExtra = result.extra > 0
            ? (ENCHANT_BONUS_SUPPORTED.has(enchantKey)
                ? ` §7(bonus efek script aktif: +${result.extra})`
                : ` §7(baru tampil di lore, belum ada bonus efek)`)
            : "";
          player.sendMessage(`§a[Enchant] §fBuku ${ENCHANTS[enchantKey].label} ${toRoman(level)} diberikan ke ${target.name}.${noteExtra}`);
          target.sendMessage(`§a[Enchant] §fKamu mendapat buku ${ENCHANTS[enchantKey].label} ${toRoman(level)} dari operator §e${player.name}`);
        }
      });
      return { status: CustomCommandStatus.Success };
    }
  );
  customCommandRegistry.registerCommand(
    { name: "cmd:online", description: "Menampilkan daftar player yang sedang online", permissionLevel: CommandPermissionLevel.Any, cheatsRequired: false },
    (origin) => {
      const player = origin.sourceEntity;
      if (!(player instanceof Player)) return { status: CustomCommandStatus.Failure, message: "Command ini hanya bisa dijalankan oleh player." };
      system.run(() => {
        const players = world.getAllPlayers();
        const names = players.map((p) => `§7- §f${p.name} §7[${getRank(p)}]`).join("\n");
        player.sendMessage(`§a[Online] §fTotal ${players.length} player:\n${names}`);
      });
      return { status: CustomCommandStatus.Success };
    }
  );
});

world.afterEvents.worldLoad.subscribe(() => {
  ensureObjective();
});

```

Kalau mau modifikasi:
1. Edit `scripts/main.js`
2. Ubah versi di `manifest.json` (`header.version` dan `modules[0].version`) setiap kali update, supaya Minecraft mau memuat ulang pack
3. Zip ulang folder ini (isinya langsung, bukan foldernya) dan ganti ekstensi jadi `.mcpack`

## ⚠️ Batasan yang perlu diketahui

- Level enchant di atas batas vanilla **bukan level resmi engine Minecraft** — itu murni dihitung & diterapkan lewat script sebagai bonus tambahan (damage extra, dsb). Level asli di engine tetap terkunci di angka maksimal vanilla.
- Command sengaja dinamai `/cenchant` (bukan `/enchant`) karena Minecraft Bedrock sudah punya command bawaan `/enchant` sendiri yang akan selalu menang jika nama bentrok.
- Proteksi api di market dicek berkala tiap 5 detik (bukan realtime instan) demi performa server.
- Fortune & Looting bekerja dengan menggandakan item drop di sekitar lokasi kejadian, bukan memodifikasi loot table asli — jadi ada jeda sepersekian detik.

## 📜 Lisensi

MIT — bebas dipakai, dimodifikasi, dan dibagikan ulang.

## 🙏 Kontribusi

Laporan bug atau request fitur silakan buka [Issues](../../issues).

##🥐 Credit
YT:@zii245
