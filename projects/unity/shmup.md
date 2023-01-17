---
title: Unity Shmup project
published: true
---

## Intro

This is a work in progress.  I am working on recreating an old project I made in the late 2000's - I've learned a lot since then and the tools have advanced quite a bit, so I should be able to do more in much less time.  I'm writing down rough notes about what I'm doing and will transcribe this into individual posts with more detail later.



FYI: I'm using Unity editor 2021.3.2f1.  The editor is constantly improving and changing, so if you are using a different version some things may be different for you.

# Part 1 - Setting Up

- Make new project, use 2D template
- Add a new object -> 2D -> Physics -> Dynamic Sprite
  - Name it Player - it's a circle now, but we'll fix that later
  - Change gravity to 0
- Using new input method
  - https://gamedevbeginner.com/input-in-unity-made-easy-complete-guide-to-the-new-system/ explained how to enable new input system
  - Install the package (in package editor, make sure to show from the right source)
  - Add the object to the project
  - Add player input to player
  - //TODO: Is there a way to automate part of this?
- Add Player script
  - Speed variable
  - Hook up to OnMove and OnFire methods
  - Move the transform in OnMove
- Improve things in Player script
  - Get reference to RigidBody2d
  - Move rb instead of directly editting transform

```csharp
public class PlayerScript : MonoBehaviour
{
    public float Speed = 3;

    private Rigidbody2D rb;
    private bool isFiring = false;

    void Start()
    {
        rb = GetComponent<Rigidbody2D>();
    }

    void OnMove(InputValue value)
    {
        rb.velocity = value.Get<Vector2>() * Speed;
    }
}
```

# Part 2 - The Shooting Part of Shmup

- Made a bullet sprite
  - Same dynamic sprite default
  - 0.75x0.1 scale, yellow color
  - 0 gravity
  - Saved it as a prefab

- Add shooting logic to player script
  - Made reference for bullet
  - Instantiated the bullet, set the velocity and position

```csharp
public GameObject Projectile;

public float BulletSpeed = 15;
public float BulletLifetime = 5;

void Update()
{
    if (TimeTilShot > 0) TimeTilShot -= Time.deltaTime;
}
void OnFire(InputValue value)
{
    var newBullet = Component.Instantiate(Projectile, this.transform.position, this.transform.rotation);
    var newBulletRb = newBullet.GetComponent<Rigidbody2D>();
    if (newBulletRb != null)
    {
        newBulletRb.velocity = newBullet.transform.right * BulletSpeed;
    }
    //Otherwise a bullet that misses will go on forever, taking up CPU for no reason
    Destroy(newBullet, BulletLifetime);
}
```
- Bullet is pushing the player!
  - Created Allies and Enemies layers
  - Disabled collision in Physics 2D options

## Better Weapons
Things I want to add:
- Constant firing
- Inaccurate firing
- Eventually: multiple weapons!

Oof!
- This is too much for the player script - make a weapon script
  - We'll have a lot, so go ahead and make a scripts folder
  - Make a new empty object on the player - this will be our weapon
    - Put it at the front of the player - you'll see why later
  - Add WeaponScript to that weapon
    - Move things from PlayerScript to WeaponScript
    - Add a reference to WeaponScript from PlayerScript
    - For convenience, switch fireRate to be expressed as shots per second


```csharp
//PlayerScript
public WeaponScript Weapon;
private bool isFiring = false;

void OnFire(InputValue value)
{
    isFiring = value.Get<float>() > 0;
}

//WeaponScript
public GameObject Projectile;

public float BulletSpeed = 15;
public float BulletLifetime = 5;
public float BulletAngle = 5;

public float ShotsPerSecond = 200;
private float TimeBetweenShots => 60f / ShotsPerSecond;
private float TimeTilShot = 0f;

void Update()
{
    if (TimeTilShot > 0) TimeTilShot -= Time.deltaTime;
}

public void TryFire()
{
    //Why "while" instead of "if", you ask?
    //In case of very high fire rates, we may fire multiple projectiles in one frame
    while (TimeTilShot <= 0 && Projectile != null)
    {
        TimeTilShot += TimeBetweenShots;
        var angleDiff = Random.Range(-BulletAngle, BulletAngle);
        var rotation = Quaternion.Euler(0, 0, this.transform.rotation.eulerAngles.z + angleDiff);

        var newBullet = Component.Instantiate(Projectile, this.transform.position, rotation);
        var newBulletRb = newBullet.GetComponent<Rigidbody2D>();
        if (newBulletRb != null)
        {
            newBulletRb.velocity = newBullet.transform.right * BulletSpeed;
        }
        Destroy(newBullet, BulletLifetime);
    }
}
```

# Part 3 - The bullets need something to hit!
- Create some walls
  - Create a couple static physics2d sprites
  - Stretch one vertically, put at the right edge
  - Stretch one horizontally, put at the bottom
- Run the game - the bullets hit the wall and kind of pile up...
- Enter, the DestructibleScript!

```csharp
public class DestructibleScript : MonoBehaviour
{
    //For visual effects, explosions, shrapnel, bouncing bullets, you name it!
    public GameObject[] SpawnOnDeath;

    private void OnDestroy()
    {
        if (SpawnOnDeath?.Length > 0)
        {
            foreach (var obj in SpawnOnDeath)
            {
                Instantiate(obj, transform.position, transform.rotation);
            }
        }
    }

    private void OnCollisionEnter2D(Collision2D collision)
    {
        Destroy(this.gameObject);
    }
}
```
- This is a fun time to play with particle effects!
  - I made some sparks that fly out when hitting the walls

# Part 4 - Missiles!
- I just can't help myself.  Any time I make a game, I want to make more and more weapons
- Things we need for missiles to work:
  - Smoke trails
  - Explosions
  - Multiple weapons on one player!
  - Let's play with lighting, why not?
- Create a new Physics2d dynamic sprite, named Missile
  - Stretch it like with the bullet, but make it grey
  - Add child Particle System, slide to left side of missile
    - Adjust it so it emits over time
    - Tweak the visuals until it looks good
    - Add explosion particle system, add it as the spawn on death
  - Save all that as prefabs
- Tweak the PlayerScript:
```csharp
private WeaponScript[] Weapons;
void Start()
{
    rb = GetComponent<Rigidbody2D>();
    //Because I don't want to have to add them manually each time
    Weapons = GetComponentsInChildren<WeaponScript>();
}

void Update()
{
    if (Weapons?.Length > 0 && this.isFiring)
    {
        foreach (WeaponScript weapon in Weapons)
        {
            weapon.TryFire();
        }
    }
}
```
- Add a new weapon in the editor
  - New game object under Player
  - Add script to it and set its properties (including giving it a projectile)

Test it out and hooray, it works!  But wait, the smoke trails suddenly disappear when the missile hits!  Lots of things will have trailing particles, so let's add this to things that will die - aka, the DestructibleScript

```csharp
public GameObject[] SpawnOnDeath;
// In case we want to sometimes *not* detach particle systems later
public bool ShouldDetachParticles = true;

// Other code omitted for brevity

private void OnCollisionEnter2D(Collision2D collision)
{
    if (ShouldDetachParticles)
    {
        //Grab all the particle systems we have attached
        foreach (var ps in GetComponentsInChildren<ParticleSystem>())
        {
            //Detach from its parent (me)
            ps.transform.SetParent(null, true);

            //If the parent was scaled, particles will scale weirdly without this
            ps.transform.localScale = Vector3.one;

            //Stop emitting particles so the whole system will die when finished
            ps.Stop();
        }
    }
    Destroy(this.gameObject);
}

void OnDestroy()
{
    foreach (var ps in GetComponents<ParticleSystem>())
    {
        //Disconnect from parent, stay in same world position
        ps.transform.SetParent(null, true);

        //Stop emitting more particles
        ps.Stop();
    }
}
```
- Missiles have thrust, so I want there to be acceleration, not just velocity!
- RigidBody has no built-in acceleration, so we'll make a script!
```csharp
public class AcceleratorScript : MonoBehaviour
{
    public Vector2 Acceleration;
    private Rigidbody2D rb;

    void Start()
    {
        this.rb = GetComponent<Rigidbody2D>();
    }

    void Update()
    {
        if (rb != null)
        {
            rb.velocity += Acceleration * Time.deltaTime;
        }
    }
}
```
- Set the X value to like 20, change the missile weapon to have a really low (like 1) projectile speed.  Looks great!
- But wait: change shooting angle of the missiles to be wide and watch what happens - the thrust is ALWAYS to the right :(
  - Also, velocity is set in weapon, why not set acceleration there too?

```csharp
//At top of script
public float BulletSpeed = 15;
public float BulletAcceleration = 0;

//inside TryFire();
newBulletRb.velocity = newBullet.transform.right * BulletSpeed;
if (BulletAcceleration != 0)
{
    //This way we don't have to add it in the editor for every accelerating projectile!
    var newBulletAccel = newBullet.AddComponent<AcceleratorScript>();
    newBulletAccel.Acceleration = newBullet.transform.right * BulletAcceleration;
}
```
- Remove the script from missile, change the values in the missile weapon and run it!

# Part 5 - Enemies

- Add new 2D Object -> Physics -> Dynamic Sprite
  - Name it enemy
  - Click sprite and change it from Circle to Square
  - Change to square collider
  - Make it a nice evil color... I picked purple 😬
  - Set gravity to 0
  - Change layer to "Enemies"
  - Add a child object
    - Call it Weapon
    - Add weapon script
    - Add *another* Dynamic Sprite
      - Name it Enemy Bullet
      - Make it a similar evil color... I picked pink
      - Also 0 gravity
      - Add DestructibleScript
      - Change layer to Enemies!!
      - Save as a prefab
    - Add prefab (not the one in the scene) bullet to weapon
    - Set values of enemy weapon really low - we don't want it shooting nearly as much as the player!
  - Save Enemy as prefab, then make a few copies around the screen
- Run it and kill some enemies! 😈
  - Whoops, I forgot to add DestructibleScript to Enemy.  It's fun to knock them around, but that's not what I wanted.
- Add new EnemyScript to Enemy
  - Add reference to the Weapon
  - Let's make our enemies dumb for now - just have them fire their weapon's constantly
  - If it looks really familiar, it's because I stole most of it from PlayerScript

```csharp
public class EnemyScript : MonoBehaviour
{
    public WeaponScript[] Weapons;
    //We'll use this later, when our enemies get some AI
    private bool isFiring = true;

    void Start()
    {
        Weapons = GetComponentsInChildren<WeaponScript>();
    }

    void Update()
    {
        if (Weapons?.Length > 0 && this.isFiring)
        {
            foreach (WeaponScript weapon in Weapons)
            {
                weapon.TryFire();
            }
        }
    }
}
```
- Run it and... the enemies are shooting the wrong way 🤦‍♂️
- Turns out we don't need to update the WeaponScript.  We're already referencing the parent object's rotation, so just set the Enemy's Weapon child object to have a 180 Z-rotation.
- Run it again and let the bullets hit the player.  Hooray physics!  But try to shoot after you're hit, and notice your weapons firing in circles
  - Change the player image to a square and you'll see what's happening - when the bullets push you, they impart a spin!
- Easy fix: go to Player's RigidBody2D, open the Constraints section and check "Freeze Rotation".  While I'm at it, I increased the player's mass because I really don't want it getting pushed around
- All the enemies fire in tandem - that's lame.
  - Adjust the weapon script for some random initialization
```csharp
void Start()
{
    TimeTilShot = Random.Range(0, TimeBetweenShots);
}
```
- Alright, instantly dying enemies is pretty boring.  Time to add HP to the mix!  We'll start in DestructibleScript.  This is mostly just rearranging what we've got already.
  - Add some variables to set and track HP
  - Rather than dying as soon as a collision happens, we TakeDamage().  For now, everything will do 1 damage.
  - All the pre-OnDestroy code will go into Die()
  - When our HP drops to 0 we call Die().
```csharp
public float MaxHP = 0f; //instant death by default
private float CurrentHP;

void Start()
{
    CurrentHP = MaxHP; //start at max HP
}

private void OnDestroy() { ... } //this didn't change

private void TakeDamage(float damage)
{
    CurrentHP -= damage;
    if (CurrentHP <= 0)
    {
        Die();
    }
}
private void Die()
{
    if (ShouldDetachParticles) { ... } //this is all the same, just located in Die() instead
    Destroy(this.gameObject);
}

private void OnCollisionEnter2D(Collision2D collision)
{
    TakeDamage(1); //we'll change this soon
}
```
- Set the enemies to more than 0 HP, run it, make sure it all works
  - Once again, enemies are getting knocked around too much.  Let's think about these mass values
  - Set bullet and missile masses much lower and increase the enemies
- Alright, the enemies get hurt and die - AWESOME!  There's just two problems:
  - Bullets and missiles do the same amount of damage
  - Missile explosions don't do area-of-effect damage like you'd expect
- We have a script for *taking* damage, let's make one for *dealing* damage and look for it in DestructibleScript
```csharp
//Our new script, not much here
public class DamageDealerScript : MonoBehaviour
{
    public float Damage = 1.0f;
}

//Inside of DestructibleScript
private void OnCollisionEnter2D(Collision2D collision)
{
    var damager = collision.gameObject.GetComponent<DamageDealerScript>();

    TakeDamage(damager?.Damage ?? 1); //default to 1 in case there is no damage script
}
```
- Now the annoying part - add this to Bullet, EnemyBullet, Missile, Enemy (we shouldn't be running into it, after all), and Player (but if we do run into things, let's make it hurt!)
- I just noticed: bullets can hit each other!  Enemy bullets have managed to stop mine!  Let's fix this:
  - Add 2 new layers: Allies Projectiles and Enemy Projectiles
  - Change Bullet and EnemyBullet to these new layers
  - Go to the Physics menu and disable collisions between Enemies and their projectiles, Allies and their projectiles, and *any* projectiles with each other
- Let's make an area of effect object for our missiles:
  - Make a new AreaOfEffectScript
  - Define a min and max damage - things closer to the center will get max damage, ones further away may just get the minimum
  - Define a radius that defines how big to make the AoE effect
  - Define the point at which damage starts to fall off due to distance.  We'll do this as a percentage and default it to 25% (i.e.: if you're 25 feet from a 100-foot explosion, you still get max damage, it lowers as you go out from there)
  - NOTE: I added some code to account for different "triggers".  I omitted it because it's not needed for now
  - Here's the final product:
```csharp
public class AreaOfEffectScript : MonoBehaviour
{
    public LayerMask LayerToHit; //we need this since AOE is doing collisions manually below

    public float Radius = 1f;
    [Range(0f, 1f)]
    public float DamageFalloffPercentage = 0.25f;
    private float DamageFalloffRadius => Radius * DamageFalloffPercentage;

    public float MinDamage = 1f;
    public float MaxDamage = 1f;
    private float DamageRange => MaxDamage - MinDamage;

    private void OnCollisionEnter2D(Collision2D collision)
    {
        //This is separate - in case we want to trigger it other ways later
        DoDamageInArea();
    }

    public void DoDamageTo(DestructibleScript target)
    {
        if (MinDamage == MaxDamage)
        {
            target.TakeDamage(MinDamage);
            return;
        }

        float distance = Vector3.Distance(target.transform.position, this.transform.position);
        if (distance < DamageFalloffRadius)
        {
            //It's within the max damage radius, so it gets the full effect
            target.TakeDamage(MaxDamage);
        }
        else
        {
            //otherwise the damage decreases to minimum for things further away
            float damagePercent = GetDamagePercent(distance);
            float damage = MinDamage + damagePercent * DamageRange;
            target.TakeDamage(damage);
        }
    }
    private float GetDamagePercent(float distance)
    {
        //Basically, how far are we from the edge of the AOE, as a percent
        //ie: 50% means we're halfway between the inner (full damage) radius and outer radius
        //so we should do halfway between min and max damage
        float distFromMaxDamage = distance - this.DamageFalloffRadius;
        return 1 - distFromMaxDamage / Radius;
    }

    public void DoDamageInArea()
    {
        foreach (var other in Physics2D.OverlapCircleAll(transform.position, Radius, LayerToHit))
        {
            var otherDestro = other.GetComponent<DestructibleScript>();
            if (otherDestro != null)
            {
                DoDamageTo(otherDestro);
            }
        }

        //In case there are multiple collisions in one frame, this prevents multi-hitting with one AOE
        Destroy(this);
    }
}
```
- It's hard to tell if this is working, so I want to add one more thing: flashing red when hit!
- This will go on the DestructibleScript, since that's for things that take damage
  - Define a color we will flash to - I used red
  - Determine how long (in seconds) the flash will last
  - Have a variable to track the original color and our progress towards finishing the flash
  - Maintain a reference to the SpriteRenderer so we can set its color
  - Turn red when we get hit
  - Smoothly transition back to our normal color
  - Here's the stuff I added:

```csharp
public class DestructibleScript : MonoBehaviour
{
    //...other variables already here
    public Color FlashColor = Color.red;
    public float FlashLength = 0.25f;
    private SpriteRenderer Sprite;
    private float FlashTransitionTime = 1f;

    void Start()
    {
        CurrentHP = MaxHP; //start at max HP
        this.Sprite = GetComponent<SpriteRenderer>();
        NormalColor = Sprite.color;
    }
    void Update()
    {
        //once we get to 1 we're back to normal
        if (FlashTransitionTime < 1f)
        {
            //This will slide us from 0 to 1, and make a smooth color transition
            FlashTransitionTime += Time.deltaTime / FlashLength;
            Sprite.color = Color.Lerp(FlashColor, NormalColor, FlashTransitionTime);
        }
    }
    public void TakeDamage(float damage)
    {
        CurrentHP -= damage;
        if (CurrentHP <= 0)
        {
            Die();
        }
        else
        {
            FlashTransitionTime = 0f; //start flashing back over, as long as we aren't dead
        }
    }
```
- One last test and... it all works!  (at this point, I disabled the minigun and played around with the missiles... A LOT)
- Whew!  That was a lot for one session, but we're done for now

# Tangent 1 - Lights!!
- First, have to install Universal Render Pipeline - https://docs.unity3d.com/Packages/com.unity.render-pipelines.universal@14.0/manual/Setup.html
- Next, regenerate your C# project so new URP stuff shows up: https://forum.unity.com/threads/solved-is-there-a-way-to-force-unity-to-regenerate-csproj-files.868147/
- Also had to change everything to the new sprite-lit - this is annoying, so it's good that we're doing this now and not when we have 10,000 prefabs
- Add some lights and shadow casters:
  - A global light - can lower the intensity to see other lights better, but eventually it'll be pure white and 1.0 intensity
  - I added a dim red light to the back of my missile object
  - Add shadow casters to the enemies and the player
  - For the missile's spawn-on-death, I added a big red-orange light.  But now it stays forever when the missile dies - it's scripting time!
- Our script will make the light's radius decrease over time and then remove it when it hits 0.  It's pretty simple, but it'll get used all over the place for impact effects.  Here's the final product:
```csharp
using UnityEngine;
using UnityEngine.Rendering.Universal; //this is where Light2D comes from
//If Intellisense doesn't show it, you need to regenerate the C# project!

public class ShrinkingLightScript : MonoBehaviour
{
    public float StartSize = 1f;
    public float ShrinkTime = 1f;

    private Light2D light;
    private float t;

    void Start()
    {
        light = gameObject.GetComponent<Light2D>();
        t = 0;
    }

    void Update()
    {
        t += Time.deltaTime / ShrinkTime;
        if (t > 1)
        {
            Destroy(this.gameObject);
        }
        else
        {
            light.pointLightOuterRadius = Mathf.Lerp(StartSize, 0, t);
        }
    }
}
```
- And now the obligatory playing around with my creation instead of working more 😅

# Part 6 - Player Death / Respawn
- Our player doesn't die - that's a little unfair
- We could slap a destructible script on Player, but what happens when it gets removed?  Now you have nothing to control
- What exactly do we want the player to do when killed?
  - It should still spawn "on death" objects (explosions and whatnot)
  - Have a number of lives and lose one when killed
  - It should *not* be removed from the scene unless it lost its last life
  - Disappear from view for a few seconds
  - Reappear at a set spot, but be invincible for a few seconds (and appear translucent)
  - Become fully opaque and vulnerable to damage
- How do we do it?
  - Change some parts of DestructibleScript to be protected, split out into separate functions the player will need, specifically
    - Break the spawning of death objects into a new method that is called by `OnDestroy`
    - Make sure things like HP, sprite, and whatnot are `protected`, not `private`
    - Mark some methods as `virtual`
  - Extend DesctructibleScript as PlayerDestructibleScript and add the required behaviors
  - Here's the final product:

```csharp
public class PlayerDestructibleScript : DestructibleScript
{
    public int LivesLeft = 5;
    public float RespawnTime = 3f;
    public float RespawnImmuneTime = 5f;
    public Color ImmuneColor = new Color(1, 1, 1, 0.5f);
    public Vector2 RespawnPosition = Vector2.zero; // set this in the editor
    Collider2D collider;

    float respawnT = 0f;
    float immuneT = 0f;

    protected override void Start()
    {
        base.Start();
        
        collider = GetComponent<Collider2D>();
    }

    protected override void Update()
    {
        if (immuneT > 0f)
        {
            immuneT -= Time.deltaTime;
            if (immuneT <= 0f)
            {
                BecomeVulnerable();
            }
        }
        else if (respawnT > 0f)
        {
            respawnT -= Time.deltaTime;
            if (respawnT <= 0f)
            {
                Respawn();
            }
        }
        else
        {
            base.Update(); // this will stop the base from updating the sprite color
        }
    }

    protected override void Die()
    {
        SpawnDeathObjects();
        LivesLeft--;

        if (LivesLeft <= 0)
        {
            base.Die();
        }
        else
        {
            Despawn();
        }
    }

    protected void BecomeVulnerable()
    {
        Sprite.color = Color.white;
        collider.enabled = true;
    }

    protected void Despawn()
    {
        respawnT = RespawnTime;
        FlashTransitionTime = 1f; //so the sprite isn't flashing red when you respawn
        Sprite.color = Color.clear;
        collider.enabled = false;
        CurrentHP = MaxHP;
    }
    protected void Respawn()
    {
        immuneT = RespawnImmuneTime;
        Sprite.color = ImmuneColor;
        this.transform.position = RespawnPosition;
    }
}
```

# Part 7 - Player Lives / Game Over
- Player has lives, but there are a few problems
  - There's no indicator of how much health you have
  - There's no indicator of how many lives you have
  - When you run out of lives, you just disappear and the game keeps going
- Sounds like it's GUI time!
  - Add a new game object -> UI -> TextMeshPro Text.  Click the "Import TMP Essentials" option that appears, since it's our first time with TextMeshPro
  - Double-click the object to zoom to the canvas - it'll appear up and away from our game objects
  - Name our text "Player Text" - we'll be lazy for now and put all the text in one spot
  - Anchor it to the bottom - hold alt+shift and click the bottom option so it stretches the whole width
  - Add some things to the PlayerDestructibleScript:
    - Reference to the TextMeshPro text
    - A method to update that text
    - Call that method when getting hit, dying, respawning
- Here are the changes:

```csharp
public class PlayerDestructibleScript : DestructibleScript
{
    /* ... */
    public TMPro.TMP_Text PlayerText;

    protected override void Start()
    {
        /* ... */
        UpdateUI();
    }

    public override void TakeDamage(float damage)
    {
        base.TakeDamage(damage);

        UpdateUI();
    }

    protected override void Die()
    {
        /* ... */
        UpdateUI();
    }

    void UpdateUI()
    {
        if (PlayerText != null)
        {
            PlayerText.text = $"HP: {CurrentHP} / {MaxHP} | Lives: {LivesLeft}";
        }
    }
}
```
- When you lose your last life nothing happens.  Let's make a game over screen!
  - Add a new panel to the UI - call it GameOverPanel - make it black and 50% transparency.  Add the CanvasGroup component to it; that's what lets us change transparency of the whole panel at once
  - Add a new textmeshpro text to that panel - change it to say "GAME OVER", increase the font size, center it in the panel
  - Add a reference to the panelgroup in the GameManagerScript
  - Add a reference to GameManagerScript in PlayerDestructibleScript - later we'll make it so *every* GameObject has a reference to it, but let's worry about that later

```csharp
public class GameManagerScript : MonoBehaviour
{
    public CanvasGroup gameOverGroup;

    void Start()
    {
        if (gameOverGroup != null)
        {
            gameOverGroup.alpha = 0;
        }
    }

    public void ShowGameOverScreen()
    {
        if (gameOverGroup != null)
        {
            gameOverGroup.alpha = 1;
        }
    }
}

public class PlayerDestructibleScript : DestructibleScript
{
    GameManagerScript gm; //TODO: Move this to base class later
    /* ... */

    protected override void Start()
    {
        /* ... */

        gm = FindObjectOfType<GameManagerScript>(); //TODO: Move this to base class later
    }

    protected override void Die()
    {
        /* ... */
        if (LivesLeft <= 0)
        {
            if (gm != null) gm.ShowGameOverScreen();
            Time.timeScale = 0.5f; // for some dramatic flair

            base.Die();
        }
        /* ... */
    }

    /* ... */
}
```
- Now let's add some cheats for us to make it easier to test things.  In our DevModeScript we will add:
  - A way to spawn enemies
  - A way to increase/decrease our lives count
  - Ways to speed up / slow down time
  - *Later we'll use this for all kinds of things*
- Step 1 - make the input changes
  - Open input actions
  - Make a new action map called DevMode
  - Add some actions:
    - ChangeLives - 1D Axis, I bound to num+/-
    - ChangeSpeed - 1D Axis, I bound to pgup/pgdn
    - SpawnEnemies - button, I bound to num *
- Step 2 - add input to the camera where the devmode script is located.  Point it at the InputActions, set map to the DevMode action map we made above
- Step 3 - Create the script:
```csharp
public class DevModeScript : MonoBehaviour
{
    public GameObject enemyPrefab;
    PlayerDestructibleScript[] players;

    void Start()
    {
        players = FindObjectsOfType<PlayerDestructibleScript>() ?? new PlayerDestructibleScript[] { };
    }

    void OnChangeLives(InputValue value)
    {
        if (value.Get<float>() > 0)
        {
            foreach (var player in players) player.LivesLeft++;
        }
        else
        {
            foreach (var player in players) player.LivesLeft--;
        }
    }

    void OnChangeSpeed(InputValue value)
    {
        if (value.Get<float>() > 0)
        {
            Time.timeScale += 0.25f;
        }
        else
        {
            Time.timeScale -= 0.25f;
        }
    }

    void OnSpawnEnemies(InputValue value)
    {
        Vector3 position = Random.insideUnitCircle;

        var enemy = Instantiate(enemyPrefab);
        enemy.transform.position = position;
    }
}
```
- Step 4 - Assign enemy to `enemyPrefab` in the script
- Try it and... only one of my maps work.  Looked into it, turns out maps are not meant to be used this way.  Rather than fight it to make it work for a debug script, I just copied the actions from the DebugMode action to the player action and put the script on the player.  Now it all works!  Hooray!

# Part 8 - Visual Upgrades
The game looks pretty.... terrible.  Nobody wants to play a game about a circle that shoots other circles at squares.  Time to make some visual upgrades so that things look better.

## Parallax Background
It doesn't look like we're flying because there's no real movement on the screen.  We'll fix that with a scrolling background!!

- First, get a couple repeating backgrounds.  You want one that is a solid image (ie: black with white dots for stars) and others that have transparency (ie: transparent with white dots).  I made a couple backgrounds myself and stuck them in a folder called `Backgrounds`.
- In Unity, select all your background you made (in the assets list) and...
    - Change `Texture Type` to `Default`
    - Set `Wrap Mode` to `Repeat`
    - 

Ran into a few problems while trying to do this:
- All my images are fuzzy
- Particles seem to disappear when they're in front of my bg textures
- I made my images too tiny, which makes the fuzziness even worse

//TODO: Fix the background problems - look at an old project for inspo

## "Corpses" that fall
Enemies exploding is cool, but we can make it cooler

- Make a new GameObject -> Spriite -> Square, name it `Enemy Corpse`
    - Change color to darker than actual enemy
    - Add a RigidBody2d, set gravity scale to 0.5
    - Add a Box Collider 2D, leave settings alone
    - Change layer to new layer called `Corpses`
    - Save it as a prefab
- Make sure Corpses only collide with the ground
    - Edit -> Project Settings -> Physics 2D -> Uncheck anything under `Corpses` except for `Default`.  Eventually we'll make a Terrain layer so it's even more explicit
- Add EnemyCorpse to the "Spawn On Death" part of the Enemy prefab
- Hit Play and watch as corpses fall when the enemies die!
- For fun, let's add some smoke to our corpses
    - Add new GameObject -> Effects -> Particle System to the Enemy Corpse prefab
    - Change it around to look like fire turning to smoke.  Make sure simulation space is World, otherwise it looks weird as the corpses move
- NOTE: Things don't look exactly right because the corpses aren't moving, but it's looking better already!


## Using real images
- Go download some images from Kenney - btw you should give him your money Kenney is great
- Problem: most of his images are pointing up, vertically
- My solution: put image on a child object, rotate that
- New problem: `DestructibleScript` can't find the SpriteRenderer component.  Solution:
    - Let's flash ***all*** the sprites on an object
    - Change from `GetComponent` to `GetComponentsInChildren` to get everything
    - Loop through to flash or otherwise update things
- Here are the parts that changed:
```csharp
/* ... */
using System.Linq; // so we get the .First() method below

public class DestructibleScript : MonoBehaviour
{
    /* ... */
    protected SpriteRenderer[] Sprites;

    protected virtual void Start()
    {
        /* ... */
        Sprites = GetComponentsInChildren<SpriteRenderer>();
    }

    protected virtual void Update()
    {
        if (FlashTransitionTime < 1f)
        {
            FlashTransitionTime += Time.deltaTime / FlashLength;
            UpdateSpriteColor(Color.Lerp(FlashColor, NormalColor, FlashTransitionTime));
        }
    }
    protected void UpdateSpriteColor(Color color)
    {
        foreach (var sprite in Sprites)
        {
            sprite.color = color;
        }
    }
    /* ... */
}
```
- Why put it in a separate method?  Remember that `PlayerDestructibleScript` extends this and references the sprite, so we'll have to fix it as well.  Any mention of `Sprite.color` can be changed to call `UpdateSpriteColor()` instead.

## Pauses and shakes on major events (deaths, mainly)
One way to juice up a game is to have very slight, almost unnoticeable pauses during major events and totally noticeable shaking camera.  This is something we'll want many different objects to do, so we'll use our `GameManagerScript` we made before.

First, let's get an example of the end result:

- In `GameManagerScript`:
```csharp
public class GameManagerScript : MonoBehaviour
{
    private Camera cam;
    private Vector3 camOrigin;
    private float shakeT = 0f;
    private float shakeLength = 0f;
    private float shakeIntensity = 0f;

    public CanvasGroup gameOverGroup;

    void Start()
    {
        if (gameOverGroup != null) { /* ... */ }

        cam = Camera.main;
        if (cam != null)
        {
            camOrigin = cam.transform.localPosition;
        }
    }

    void Update()
    {
        if (shakeT > 0)
        {
            shakeT -= Time.deltaTime / shakeLength;
            if (shakeT <= 0) ResetCamera();
            else
            {
                Vector3 newPosition = Random.insideUnitSphere * (shakeIntensity * shakeT);
                cam.transform.localPosition = newPosition + camOrigin;
            }
        }
    }

    public void ShowGameOverScreen() { /* ... */ }

    public void ShakeScreen(float intensity, float length)
    {
        this.shakeIntensity = intensity;
        this.shakeLength = length;
        shakeT = 1.0f;
    }
    public void ResetCamera()
    {
        this.cam.gameObject.transform.localPosition = camOrigin;
    }
}
```
- And in `DevModeScript`, I added `FindObjectOfType<GameManagerScript>().ShakeScreen(1, 1);` in the `OnSpawnEnemies` method so I have a way to test
- Run it and make sure - everything looks great!
- When do we want the screen to shake?
    - Player dies
    - Explosions
    - Enemy dies
- Something spawns any time these happen, so let's make a `ShakeOnSpawn` script that calls the `GameManagerScript`:
```csharp
public class ShakeOnSpawnScript : MonoBehaviour
{
    GameManagerScript gm;

    public float shakeIntensity = 1f;
    public float shakeLength = 1f;

    void Start()
    {
        this.gm = FindObjectOfType<GameManagerScript>();

        if (this.gm != null && shakeIntensity > 0 && shakeLength > 0)
        {
            gm.ShakeScreen(shakeIntensity, shakeLength);
        }
    }
}
```
- Place this script on the Explosion game object and set the values to something smaller (1 second of shaking is too much).  Now the player dying and its missiles hitting make things shake!!

Next up: dramatic pauses!!  Since I am once again doing something with Lerps, I am going to make my Lerp helper:
```csharp
public class Lerper
{
    float start, end, length;
    float t = 0;
    public bool IsRunning
    {
        private set;
        get;
    } = false;

    public bool IsFinished => t >= 1;

    private Lerper() { throw new System.NotSupportedException(); }
    public Lerper(float start, float end, float length)
    {
        this.start = start;
        this.end = end;
        this.length = length;
    }

    public void Update(float delta)
    {
        if (this.IsRunning)
        {
            t += delta / length;
            if (this.IsFinished)
            {
                t = 1;
                this.IsRunning = false;
            }
        }
    }
    public static Lerper CreateAndRun(float start, float end, float length)
    {
        var lerper = new Lerper(start, end, length);
        lerper.Run();
        return lerper;
    }
    public void Run(bool resetInProgress = true)
    {
        this.IsRunning = true;
        if (resetInProgress)
        {
            this.t = 0f;
        }

    }

    public float Value => Mathf.Lerp(start, end, t);
}
```
- I went back and used this new script instead of what I had already in `ShrinkingLightScript` for example.
    - Replace `float t;` with `Lerper lightLerp;`, init it in the constructor as `lightLerp = Lerper.CreateAndRun(StartSize, 0, ShrinkTime);` and change `Update()` to read like so:
```csharp 
void Update()
{
    lightLerp.Update(Time.deltaTime);
    if (lightLerp.IsFinished)
    {
        Destroy(this.gameObject);
    }
    else
    {
        light.pointLightOuterRadius = lightLerp.Value;
    }
}
```
- Simple idea, right?  We'll use it in our pause script as well.  Our script will...
    - Pause for an amount of time (if called by another object)
    - Smoothly return to normal speeds
- Here's the final result:
```csharp
public class GameManagerScript : MonoBehaviour
{
    private Lerper pauseLerper;
    private float pauseLength = 0;

    /* ... */

    void Start()
    {
        /* ... */
        
        pauseLerper = new Lerper(0, Time.timeScale, 0.1f);
    }

    void Update()
    {
        // I split this into 2 separate methods so Update() isn't junked up
        UpdateShake();
        UpdateDramaticPause();
    }
    void UpdateShake() { /* ... */ } // This has everything Update() had before
    void UpdateDramaticPause()
    {
        if (pauseLength > 0)
        {
            pauseLength -= Time.unscaledDeltaTime;
            Time.timeScale = 0f;
        }
        else if (pauseLerper != null && pauseLerper.IsRunning)
        {
            pauseLerper.Update(Time.unscaledDeltaTime);
            Time.timeScale = pauseLerper.Value;
        }
    }

    public void ShowGameOverScreen() { /* ... */ }
    public void ShakeScreen(float intensity, float length) { /* ... */ }
    public void ResetCamera() { /* ... */ }

    public void DramaticPause(float length)
    {
        Time.timeScale = 0f;
        this.pauseLength = length;
        this.pauseLerper.Run();
    }
}
```
- I tested this and played with values by adding a call to the gm in `DevModeScript`.  I settled on a very short 0.05f as the pause length that would make sense.
- Add some code to our Player to call the gm and do a Dramatic Pause when the player is hit and an even bigger one when the player dies.  We've already got a reference to `GameManagerScript` in `PlayerDestructibleScript`, so this is VERY easy.  Just add `gm.DramaticPause(0.02f);` at the top of `TakeDamage()` and a longer pause (like 0.2f, still very short) at the top of `Die()`.
- Test it out, make sure all is well... hooray!  Tweak the values 'til it feels right to you, but we're done!

# Part 9 - Main Menu
Right now, our game starts and you're immediately in the game.  When you die, it just sits at the GAME OVER screen and nothing happens.  Obviously, we need a menu!

- Save the scene I'm in - it's still just called "SampleScene", and that's okay for now
- Create a new scene called "MainMenu"
- Add some TextMeshPro text with the game's title and a TMP Button that says "Play"
    - NOTE: This is just like how we made the UI for the game itself
    - For the TMP Button, you can change the text by changing the nested Textbox object!
- Style and color things to your heart's content.  Mine will look plain, but there's no reason yours has to!

## Responding to UI button clicks

- Make sure you go to the EventSystem object and click the button to swap to the new Input system
- Add a script to EventSystem while you're there called `MainMenuScript`
```csharp
using UnityEngine;
// Make sure to include UnityEngine.SceneManagement!
using UnityEngine.SceneManagement;

public class MainMenuScript : MonoBehaviour
{
    public void StartGame()
    {
        SceneManager.LoadScene("SampleScene"); // if you named your scene, change this
    }
}
```
- Click your "Play Game" button
- Under Button, see the `On Click()` box?  Click the plus sign
    - Add the object you attached your script to
    - Select the MainMenuScript/StartGame option
- Test it out!

## Game Over menus
So we start the game, and now we have no way to get back.  We already have a *game over* UI, putting a button there will be easy

- Let's reuse our `MainMenuScript` - add it to the `EventSystem` object in this scene as well
- Add a method to it that we'll need later
```csharp
public void LoadMainMenu()
{
    SceneManager.LoadScene("MainMenu");
}
```
- I bet you can see where I'm going with this.  We're going to give our player a button to click when they die that takes them back to the main menu!
- Add a couple of buttons to the "Game Over Panel" - remember that its Alpha (in the Canvas Group component) is set to 0 so you can't see it
    - A red button that says "Quit" and calls the `LoadMainMenu()` method
    - A green button that says "Restart" and calls the `StartGame()` method
- Run the game and give it a shot.  Couple problems:
    - On-death explosions happen when the scene reloads.  We'll deal with that another time.
    - Quit button doesn't work - the console has an error explaining why.  Let's fix that now
- Open our MainMenu scene again *(we should've done this earlier)*
    - Go to File -> Build Settings -> Add open scenes
- Try running it again... success!!

## Pause menu

I, for one, am tired of ramming my ship into things just to get back to the main menu.  Let's make a pause menu that lets use go back to the main menu!

- Straight up copy/paste the `GameOverPanel`, and...
    - Rename it to `PausePanel`
    - Change the "RESTART" button to say "RESUME" instead
- Quit game is already wired up, but we need a resume method for that button to call.  I put it in the GameManager, since it already deals with time for dramatic pauses.  Here's what I added:
```csharp
public class GameManagerScript : MonoBehaviour
{
    public bool IsPaused { get; private set; }

    /* ... */

    void Start()
    {
        /* ... */

        IsPaused = false;
    }

    void Update()
    {
        if (!IsPaused)
        {
            // Otherwise if you pause during a shake it'll go foreever
            UpdateShake();

            // This way we won't accidentally unpause
            UpdateDramaticPause();
        }
    }

    /* ... */

    public void PauseGame()
    {
        IsPaused = true;
        Time.timeScale = 0;
    }
    public void ResumeGame()
    {
        IsPaused = false;
        Time.timeScale = 1f;
    }
}
```
- Now to reference our `ResumeGame()` method on our resume button
- We need a way to call `PauseGame()` from the Player.  Let's update our PlayerScript.  One thing to note: we had a reference to GameManager in PlayerDestructibleScript, but not in PlayerScript - we'll fix this next in a second tangent
```csharp
public class PlayerScript : MonoBehaviour
{
    /* ... */
    GameManagerScript gm;

    void Start()
    {
        /* ... */

        gm = FindObjectOfType<GameManagerScript>();
    }

    /* ... */

    void OnPause(InputValue value)
    {
        // This way we can unpause by hitting Esc again
        if (value.Get<float>() > 0)
        {
            if (gm.IsPaused) gm.ResumeGame();
            else gm.PauseGame();
        }
    }
}
```
- We need to add the pause functionality to our input manager next
    - Open the InputActions file
    - Add an Action to the Jet input
    - Set the binding to the Escape key
- Run it and make sure it works... okay, so the Pause Menu is there forever, but hitting the Esc key pauses and clicking Resume works just fine.  Whoohoo!!
- Alright, the pause menu needs to start out invisible and only appear when we pause the game.  We need to add a little bit to our pause script.  We need to...
    - Add a reference to the pause `CanvasGroup` (and reference it in the Unity editor)
    - Make sure the Pause Panel is hidden at the start of the game
    - Show it when pausing
    - Hide it when unpausing
```csharp
/* ... */
public CanvasGroup pauseGroup;

void Start()
{
    /* ... */

    if (pauseGroup != null)
    {
        pauseGroup.alpha = 0;
    }
}

/* ... */

public void PauseGame()
{
    if (pauseGroup != null)
    {
        pauseGroup.alpha = 1;
        IsPaused = true;
        Time.timeScale = 0;
    }
}
public void ResumeGame()
{
    if (pauseGroup != null)
    {
        pauseGroup.alpha = 0;
        IsPaused = false;
        Time.timeScale = 1f;
    }
}
```
- Everything looks fine and functions.... nope.  If you exit the game and reenter it, it gets stuck.  What's happening here?  I notice the enemy corpses (which shouldn't spawn when exiting the game, we'll fix it later) don't fall on the main menu - I bet the time scale is still 0 when we load the main menu.  Let's fix that!  Luckily it's easy, you just have to stick a `Time.timeScale = 1;` at the top of the `MainMenuScript`'s `StartGame()` and `LoadMainMenu()` methods.
- Try again... and now everything works as expected!!  Except it's all a little sudden.  Wouldn't it be better if the panels faded in and out?  Let's make a new `PanelAnimationScript` on one of our panels (any of them).  We want to:
    - Get a reference to the attached CanvasGroup (since it sets the transparency of things)
    - Have Show() and Hide() methods that set a target transparency
    - Smoothly transition in or out during the Update() method
    - Some menus we'll want to be visible right away (like the main menu) and others we'll want to be hidden at first (like the pause menu), so we'll have a `public` variable to set the starting alpha to either 1 or 0.
    - Update existing references to those `CanvasGroup`s and instead reference our `PanelAnimationScript`'s methods
- This sounds like a perfect job for our `Lerper` class, except...
    - We need to Lerp up *and* down, and the `Lerper` only go one direction.  We'll update `Lerper` to lerp in either direction and utilize that in the new script.  Here's how it went:
```csharp
public class Lerper
{
    float start, end, length;
    float t = 0;
    public bool IsRunning
    {
        private set;
        get;
    } = false;
    bool goingUp = true;

    public bool IsFinished => (goingUp && t >= 1) || (t <= 0);

    private Lerper() { throw new System.NotSupportedException(); }
    public Lerper(float start, float end, float length, bool goingUp = true, bool startAtBeginning = true)
    {
        this.start = start;
        this.end = end;
        this.length = length;

        t = startAtBeginning ? 0f : 1f;
    }

    public void Update(float delta)
    {
        if (this.IsRunning)
        {
            t += delta / length;
            if (this.IsFinished)
            {
                t = 1;
                this.IsRunning = false;
            }
        }
    }
    public static Lerper CreateAndRun(float start, float end, float length)
    {
        var lerper = new Lerper(start, end, length);
        lerper.LerpToMax();
        return lerper;
    }
    public static Lerper CreateAndRunReverse(float start, float end, float length)
    {
        var lerper = new Lerper(start, end, length);
        lerper.t = 1;
        lerper.LerpToMin();
        return lerper;
    }
    private void Run(bool resetInProgress)
    {
        this.IsRunning = true;
        if (resetInProgress)
        {
            this.t = 0;// goingUp ? 0 : 1;
        }
    }
    public void LerpToMin(bool resetInProgress = true)
    {
        goingUp = false;
        Run(resetInProgress);
    }
    public void LerpToMax(bool resetInProgress = true)
    {
        goingUp = true;
        Run(resetInProgress);
    }

    public float Value => goingUp ? Mathf.Lerp(start, end, t) : Mathf.Lerp(end, start, t);
}

public class PanelAnimationScript : MonoBehaviour
{
    public bool IsInitiallyVisible = false;
    public bool IsTransitioningAtSpawn = false;
    public float AnimationLength = 1.0f;

    public bool IgnoreTimeScale = true;

    CanvasGroup panel;
    Lerper alphaLerper;

    void Start()
    {
        panel = GetComponent<CanvasGroup>();

        bool goingUp = this.IsInitiallyVisible;
        bool startAtBeginning = !IsInitiallyVisible || goingUp;

        alphaLerper = new Lerper(0, 1, AnimationLength, goingUp, startAtBeginning);
        panel.alpha = alphaLerper.Value;
        if (IsTransitioningAtSpawn)
        {
            if (goingUp) alphaLerper.LerpToMax();
            else alphaLerper.LerpToMin();
        }
    }

    void Update()
    {
        alphaLerper.Update(IgnoreTimeScale ? Time.unscaledDeltaTime : Time.deltaTime);
        panel.alpha = alphaLerper.Value;
    }

    public void FadeIn()
    {
        alphaLerper.LerpToMax();
    }
    public void FadeOut()
    {
        alphaLerper.LerpToMin();
    }
}
```
- Try different values to see what fading speeds suit you.  I think this is enough messing around for now.


# Tangent 2 - Organization
***TODO***:
- Move scripts into a scripts folder, that type of thing
- Create a base class
- Have every other script extend that base class

I haven't been organizing anything I'm doing, so it's a bit of a mess.  In this tangent, I'm going to fix that.  Here's how I organized things:

- Backgrounds
- Effects
    - Lights
    - Particles
- Prefabs
    - Enemy
    - Player
- Scenes
- Scripts
- Sprites
    - Kenney *(these are ones we got from Kenney's game assets)*
- TextMesh Pro

I swear it used to break a bunch of things when you moved scripts and game objects.  Surprisingly, everything moved just fine.  I ran the game and it all worked exactly as it did before the move.  Notably, though, when I went to Visual Studio it was not pleased.  I closed VS, went to Unity and went to `Edit -> Preferences -> External Tools -> Regenerate project files`.  I had to re-select Visual Studio as the `External script editor` before the button to regenerate things appeared.

Now, for the cleanup that will take some effort.  We've already run into several instances where multiple scripts need to keep a reference to the Game Manager.  This is going to keep happening, and there will likely be other shared functionality we'll need, so let's put it all into a base class that all our other game objects will extend from.

```csharp
public abstract class BaseGameObject : MonoBehaviour
{
    protected GameManagerScript gm;

    protected virtual void Start()
    {
        gm = FindObjectOfType<GameManagerScript>();
    }

    protected virtual void Update()
    {
        if (!gm.IsPaused) UpdateUnpaused();
        UpdateAlways();
    }
    protected abstract void UpdateUnpaused();
    protected abstract void UpdateAlways();
}
```
- Look for any reference to `GameManagerScript` and get rid of its declaration at the top and initialization in `Start()`.  We had existing references in:
    - DevModeScript
    - PlayerDestructibleScript
    - PlayerScript
    - ShakeOnSpawnScript
- In each script (other than the Game Manager itself) change the class declaration to extend from `BaseGameObject` instead of `MonoBehavior`.  This also means you need to:
    - Add `protected override` before `Start()` and call `base.Start();` at the beginning of the method.
    - Change `Update()` to `UpdateUnpaused()` (of course with `protected override` before it)
    - Be sure to get rid of any unused `Start()` or `Update()` methods that Unity puts there by default!
    - If a class doesn't use one of those update methods, you still have to include an empty method to satisfy the base class's contract.

That's it for now.  Our base game object will make things easier later. :)

# Part 10 - Beam weapons
Bullets are cool, missiles are awesome, but it wouldn't be a sci-fi game without lasers!!  Problem is, lasers don't work quite like projectiles, so we can't reuse the existing `WeaponScript`.  First thing's first is to make a script that will handle the beam itself.  It needs to:

- Figure out how long the beam should be (i.e.: cast out from the laser until we hit something that should block it, like a wall)
- Do damage to anything hitting the beam - beams will do damage over time continuously when colliding with a thing
- Expose a method that will enable the laser (this will get called by a weapon)
- Turn the laser off again after a short time
- Draw some effects where the laser ends - I created a light and some particles
```csharp
public class BeamScript : BaseGameObject
{
    public float DamagePerSecond = 1f;
    public float lifetime = 0.1f;
    private float lifeT = 0f;

    private LineRenderer line;

    private float radius = 0.1f;
    private const float MAX_DISTANCE = 1000f;

    public GameObject[] hitEffects;
    private int damageLayers;
    public LayerMask blockingLayers;
    protected override void Start()
    {
        base.Start();

        line = GetComponent<LineRenderer>();
        radius = line.startWidth;

        damageLayers = Physics2D.GetLayerCollisionMask(this.gameObject.layer);

        FinishShooting();
    }

    protected override void UpdateAlways() { }
    protected override void UpdateUnpaused()
    {
        if (lifeT > 0)
        {
            lifeT -= Time.deltaTime;

            RaycastHit2D hit = Physics2D.CircleCast(transform.position, radius, transform.right, MAX_DISTANCE, blockingLayers);
            if (hit.collider == null)
            {
                SetLength(MAX_DISTANCE, false);
            }
            else
            {
                SetLength(hit.distance);
            }
            DoDamage(Physics2D.CircleCastAll(transform.position, radius, transform.right, hit.distance, damageLayers));
        }
        else
        {
            FinishShooting();
        }
    }

    public void StartShooting(float? beamLifetime = null)
    {
        if (beamLifetime.HasValue) this.lifeT = beamLifetime.Value;
        else this.lifeT = lifetime;
        this.line.enabled = true;
    }
    private void FinishShooting()
    {
        this.line.enabled = false;
        HideEffects();
    }
    private void SetLength(float length, bool showEffects = true)
    {
        Vector3 impact = Vector3.right * length;

        this.line.enabled = true;
        this.line.SetPosition(1, impact);
        if (showEffects)
        {
            foreach (var effect in hitEffects)
            {
                effect.SetActive(true);
                effect.transform.localPosition = impact;
            }
        }
        else
        {
            HideEffects();
        }
    }
    private void HideEffects()
    {
        foreach (var effect in hitEffects)
        {
            effect.SetActive(false);
        }
    }

    protected void DoDamage(RaycastHit2D[] hits)
    {
        foreach (var hit in hits)
        {
            var destructible = hit.collider.gameObject.GetComponent<DestructibleScript>();
            if (destructible != null)
            {
                destructible.TakeDamage(DamagePerSecond * Time.deltaTime);
            }
        }
    }
}
```
- Since I was playing around anyway, the light at the end of my laser has this little script on it:
```csharp
public class LightDancingScript : BaseGameObject
{
    public Light2D light;

    public Color color1;
    public Color color2;

    public float wiggleRadius = 0.1f;

    public float minIntensity = 0.75f;
    public float maxIntensity = 1f;

    public float minRadius = 1.9f;
    public float maxRadius = 2.1f;

    protected override void Start()
    {
        base.Start();
        light = GetComponent<Light2D>();
    }

    protected override void UpdateAlways()
    {
        if (light != null)
        {
            light.color = RandomColor.inRange(color1, color2);
            light.transform.localPosition = Random.onUnitSphere * wiggleRadius;
            light.intensity = Random.Range(minIntensity, maxIntensity);
            light.pointLightOuterRadius = Random.Range(minRadius, maxRadius);
        }
    }
    protected override void UpdateUnpaused() { }
}

public static class RandomColor
{
    public static Color inRange(Color color1, Color color2)
    {
        return new Color
        (
            Random.Range(color1.r, color2.r),
            Random.Range(color1.g, color2.g),
            Random.Range(color1.b, color2.b)
        );
    }
}
```
- I'll use this later to make some fiery glow, too.  These little, general-purpose scripts are great building blocks to making a whole game!
- Next is to make the weapon.  What I did was split `WeaponScript` up into a few pieces
    - `BaseWeaponScript` handles the concept of shooting, with delays between shots
    - `ProjectileScript` will extend the base by spawning a projectile and setting its speed
    - `BeamScript` will enable a beam rather than spawn a new projectile
- Here are the finished classes:
```csharp
public abstract class BaseWeaponScript : BaseGameObject
{
    public float FiringAngle = 5;

    public float ShotsPerSecond = 200;
    protected float TimeBetweenShots => 60f / ShotsPerSecond;
    protected float TimeTilShot = 0f;

    protected override void Start()
    {
        base.Start();

        TimeTilShot = Random.Range(0, TimeBetweenShots);
    }

    protected override void UpdateAlways() { }
    protected override void UpdateUnpaused()
    {
        if (TimeTilShot > 0) TimeTilShot -= Time.deltaTime;
    }

    public void TryFire()
    {
        while (TimeTilShot <= 0)
        {
            TimeTilShot += TimeBetweenShots;
            var angleDiff = Random.Range(-FiringAngle, FiringAngle);
            var rotation = Quaternion.Euler(0, 0, this.transform.rotation.eulerAngles.z + angleDiff);

            Shoot(rotation);
        }
    }
    protected abstract void Shoot(Quaternion rotation);
}

public class ProjectileWeaponScript : BaseWeaponScript
{
    public GameObject Projectile;

    public float BulletSpeed = 15;
    public float BulletAcceleration = 0;
    public float BulletLifetime = 5;

    protected override void Shoot(Quaternion rotation)
    {
        var newBullet = Component.Instantiate(Projectile, this.transform.position, rotation);
        var newBulletRb = newBullet.GetComponent<Rigidbody2D>();
        if (newBulletRb != null)
        {
            newBulletRb.velocity = newBullet.transform.right * BulletSpeed;
            if (BulletAcceleration != 0)
            {
                var newBulletAccel = newBullet.AddComponent<AcceleratorScript>();
                newBulletAccel.Acceleration = newBullet.transform.right * BulletAcceleration;
            }
        }
        Destroy(newBullet, BulletLifetime);
    }
}

public class BeamWeaponScript : BaseWeaponScript
{
    public BeamScript beam;
    public float BeamLifetime = 0.1f;

    protected override void Start()
    {
        base.Start();

        this.beam = GetComponentInChildren<BeamScript>();
    }

    protected override void Shoot(Quaternion rotation)
    {
        if (beam != null)
        { 
            beam.transform.localRotation = rotation;
            beam.StartShooting(BeamLifetime);
        }
    }
}
```
- Now it's just a matter of rigging things up in the editor by:
    - Fixing the existing Enemy and Player weapons, since changing the `WeaponScript` will have broken them
    - Add a laser weapon to player
        - Add `BeamWeaponScript` to it, reference a laser object with a Line2D and a `BeamScript` on it
        - Set some values - I made a very low firing angle and enough firing speed to keep the beam firing almost constantly
        - Make sure the laser is in the `Allies Projectile` layer
    - Play around with laser effects
    - Set up Physisc2D layers.  I added a new `Terrain` layer and set the wall/floor to that, then set that as the laser-stopping layer

# Tangent 3 - Spawn on Death location

It's been bugging me for a while, but the "spawn on death" objects aren't spawning in the right place.  This is especially obvious with longer projectiles, like our bullets.  Instead of spawning at the point of impact, things are spawning at the point of impact.  Let's change our script a little to fix that!

```csharp
// TODO: Write what I changed:
// - Sent position to TakeDamage
// - Changed the SpawnDeathObjects or whatever to happen in GetHurt() instead of on destroy - should stop extras from spawning
// - All the extensions of these things had to change

// Also TODO: Fix things
// - Need a bool to flag whether or not to spawn at self or impact - enemy corpses spawn at the impact point which is not cool
```



# Part 11 - Enemy AI - up to this point it's just been target practice

# Part 12 - ?

# Part 13 - ?

# Part eventually - shields that block lasers

# Tangent X - Palettes
One day my wife was watching me work on this and was like "man, wouldn't that look cool as an 80's neon color palette?"  As I went to each different object to change the colors, I thought to myself how great it would be to be able to select a color palette and have ***all*** the colors change automatically.  Bonus points if colors can be defined as modifications to the base color (e.g.: 50% darker) - basically I want to create SCSS for Unity textures.

NOTE: This may be *really* difficult to do.




# Big fat TODO list (of anything not listed above)
- Abilities
  - Shield
  - Time slowmo
  - etc
- Ship editor
  - Color picker
  - Different chassis
  - Weapon / ability selection
- Allied ships, for ambience / drama
- Campaign
  - Inter-mission map
  - Comm transmissions (portrait + textbox)
  - Predefined levels
  - Unlockables, upgrades
- Random generated levels
- Multiplayer (local)
- Limited-length levels, ship turning
- Ship landing, on-foot gameplay
  - This could be a whole other game, basically?
- Audio!
  - Sound effects
  - Music on a loop
- Use animation state machine to drive behaviors?  e.g.: player respawns, flies in from side and scales appropriately (like they're coming in from the foreground)