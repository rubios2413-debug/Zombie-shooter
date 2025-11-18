<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Zombie Shooter</title>
    <script src="https://telegram.org/js/telegram-web-app.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/phaser@3.70.0/dist/phaser.js"></script>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
            -webkit-touch-callout: none;
            -webkit-user-select: none;
            user-select: none;
        }
        body {
            margin: 0;
            padding: 0;
            overflow: hidden;
            background: #000;
            touch-action: none;
        }
        #game-container {
            width: 100vw;
            height: 100vh;
            display: flex;
            justify-content: center;
            align-items: center;
        }
    </style>
</head>
<body>
    <div id="game-container"></div>
    <script>
        // Initialize Telegram WebApp
        if (window.Telegram && window.Telegram.WebApp) {
            const tg = window.Telegram.WebApp;
            tg.ready();
            tg.expand();
            tg.disableVerticalSwipes();
            tg.isVerticalSwipesEnabled = false;
        }

        // Game Configuration
        const config = {
            type: Phaser.AUTO,
            scale: {
                mode: Phaser.Scale.FIT,
                parent: 'game-container',
                autoCenter: Phaser.Scale.CENTER_BOTH,
                width: 375,
                height: 667
            },
            physics: {
                default: 'arcade',
                arcade: {
                    gravity: { y: 0 },
                    debug: false
                }
            },
            scene: [MenuScene, GameScene, UpgradeScene, GameOverScene]
        };

        const game = new Phaser.Game(config);

        // Global Game Data
        let gameData = {
            coins: 0,
            level: 1,
            totalKills: 0,
            upgrades: {
                damage: 1,
                fireRate: 1,
                speed: 1,
                health: 3
            }
        };

        // Menu Scene
        class MenuScene extends Phaser.Scene {
            constructor() {
                super({ key: 'MenuScene' });
            }

            create() {
                this.add.text(187, 150, 'ZOMBIE SHOOTER', { 
                    fontSize: '36px', 
                    fill: '#ff0000',
                    fontStyle: 'bold'
                }).setOrigin(0.5);

                this.add.text(187, 220, `Level: ${gameData.level}`, { 
                    fontSize: '22px', 
                    fill: '#fff' 
                }).setOrigin(0.5);

                this.add.text(187, 260, `Coins: ${gameData.coins}`, { 
                    fontSize: '22px', 
                    fill: '#ffd700' 
                }).setOrigin(0.5);

                const playButton = this.add.text(187, 350, 'PLAY', { 
                    fontSize: '28px', 
                    fill: '#0f0',
                    backgroundColor: '#333',
                    padding: { x: 30, y: 15 }
                }).setOrigin(0.5).setInteractive();

                playButton.on('pointerdown', () => {
                    this.scene.start('GameScene');
                });

                const upgradeButton = this.add.text(187, 420, 'UPGRADES', { 
                    fontSize: '28px', 
                    fill: '#ff0',
                    backgroundColor: '#333',
                    padding: { x: 30, y: 15 }
                }).setOrigin(0.5).setInteractive();

                upgradeButton.on('pointerdown', () => {
                    this.scene.start('UpgradeScene');
                });
            }
        }

        // Main Game Scene
        class GameScene extends Phaser.Scene {
            constructor() {
                super({ key: 'GameScene' });
            }

            create() {
                // Game variables
                this.score = 0;
                this.wave = 1;
                this.zombiesPerWave = 5 + (gameData.level * 2);
                this.zombiesKilled = 0;
                this.playerHealth = gameData.upgrades.health;
                
                // Create player
                this.player = this.add.rectangle(187, 550, 30, 30, 0x00ff00);
                this.physics.add.existing(this.player);
                this.player.body.setCollideWorldBounds(true);

                // Create groups
                this.bullets = this.physics.add.group();
                this.zombies = this.physics.add.group();

                // Touch controls
                this.leftPointer = null;
                this.rightPointer = null;
                this.canShoot = true;
                this.autoShoot = false;

                // Enable multi-touch
                this.input.addPointer(2);

                // Touch control zones
                this.leftZone = this.add.rectangle(93, 500, 186, 334, 0x00ff00, 0.1);
                this.rightZone = this.add.rectangle(281, 500, 186, 334, 0xff0000, 0.1);

                // Add visual indicators
                this.leftArrow = this.add.text(93, 600, '←→', { 
                    fontSize: '48px', 
                    fill: '#fff',
                    alpha: 0.3
                }).setOrigin(0.5);

                this.rightIndicator = this.add.text(281, 600, 'TAP\nSHOOT', { 
                    fontSize: '24px', 
                    fill: '#fff',
                    alpha: 0.3,
                    align: 'center'
                }).setOrigin(0.5);

                // Touch event handlers
                this.input.on('pointerdown', this.handlePointerDown, this);
                this.input.on('pointermove', this.handlePointerMove, this);
                this.input.on('pointerup', this.handlePointerUp, this);

                // UI
                this.scoreText = this.add.text(10, 10, `Score: ${this.score}`, { 
                    fontSize: '16px', 
                    fill: '#fff' 
                });
                
                this.coinsText = this.add.text(10, 30, `Coins: ${gameData.coins}`, { 
                    fontSize: '16px', 
                    fill: '#ffd700' 
                });

                this.waveText = this.add.text(187, 10, `Wave: ${this.wave}`, { 
                    fontSize: '16px', 
                    fill: '#fff' 
                }).setOrigin(0.5, 0);

                this.healthText = this.add.text(365, 10, `HP: ${this.playerHealth}`, { 
                    fontSize: '16px', 
                    fill: '#f00' 
                }).setOrigin(1, 0);

                // Collisions
                this.physics.add.overlap(this.bullets, this.zombies, this.hitZombie, null, this);
                this.physics.add.overlap(this.player, this.zombies, this.hitPlayer, null, this);

                // Auto-shoot timer
                this.shootTimer = this.time.addEvent({
                    delay: 300,
                    callback: this.autoShootBullet,
                    callbackScope: this,
                    loop: true
                });

                // Spawn first wave
                this.spawnWave();
            }

            handlePointerDown(pointer) {
                if (pointer.x < 187) {
                    this.leftPointer = pointer;
                } else {
                    this.rightPointer = pointer;
                    this.autoShoot = true;
                }
            }

            handlePointerMove(pointer) {
                if (this.leftPointer && pointer.id === this.leftPointer.id) {
                    this.leftPointer = pointer;
                }
            }

            handlePointerUp(pointer) {
                if (this.leftPointer && pointer.id === this.leftPointer.id) {
                    this.leftPointer = null;
                    this.player.body.setVelocityX(0);
                }
                if (this.rightPointer && pointer.id === this.rightPointer.id) {
                    this.rightPointer = null;
                    this.autoShoot = false;
                }
            }

            update() {
                // Player movement based on left touch
                if (this.leftPointer) {
                    const speed = 250 + (gameData.upgrades.speed * 50);
                    const centerX = 93;
                    const deadZone = 20;
                    
                    if (this.leftPointer.x < centerX - deadZone) {
                        this.player.body.setVelocityX(-speed);
                    } else if (this.leftPointer.x > centerX + deadZone) {
                        this.player.body.setVelocityX(speed);
                    } else {
                        this.player.body.setVelocityX(0);
                    }
                } else {
                    this.player.body.setVelocityX(0);
                }

                // Clean up off-screen bullets
                this.bullets.children.entries.forEach(bullet => {
                    if (bullet.y < -10) {
                        bullet.destroy();
                    }
                });

                // Check if wave is complete
                if (this.zombies.countActive() === 0 && this.zombiesKilled >= this.zombiesPerWave) {
                    this.nextWave();
                }
            }

            autoShootBullet() {
                if (this.autoShoot && this.canShoot) {
                    this.shoot();
                }
            }

            shoot() {
                const bullet = this.add.rectangle(this.player.x, this.player.y - 20, 5, 15, 0xffff00);
                this.physics.add.existing(bullet);
                this.bullets.add(bullet);
                bullet.body.setVelocityY(-500);
                
                this.canShoot = false;
                const fireRateDelay = 350 - (gameData.upgrades.fireRate * 30);
                this.time.delayedCall(Math.max(100, fireRateDelay), () => {
                    this.canShoot = true;
                });
            }

            spawnWave() {
                for (let i = 0; i < this.zombiesPerWave; i++) {
                    this.time.delayedCall(i * 1000, () => {
                        this.spawnZombie();
                    });
                }
            }

            spawnZombie() {
                const x = Phaser.Math.Between(30, 345);
                const zombie = this.add.rectangle(x, 0, 25, 35, 0xff0000);
                this.physics.add.existing(zombie);
                this.zombies.add(zombie);
                
                const speed = 50 + (this.wave * 10);
                zombie.body.setVelocityY(speed);
            }

            hitZombie(bullet, zombie) {
                bullet.destroy();
                
                zombie.health = zombie.health || gameData.level;
                zombie.health -= gameData.upgrades.damage;
                
                if (zombie.health <= 0) {
                    zombie.destroy();
                    this.zombiesKilled++;
                    this.score += 20;
                    
                    const coinsEarned = Phaser.Math.Between(1, 3);
                    gameData.coins += coinsEarned;
                    gameData.totalKills++;
                    
                    this.scoreText.setText(`Score: ${this.score}`);
                    this.coinsText.setText(`Coins: ${gameData.coins}`);
                }
            }

            hitPlayer(player, zombie) {
                zombie.destroy();
                this.playerHealth--;
                this.healthText.setText(`HP: ${this.playerHealth}`);
                
                this.player.setFillStyle(0xff0000);
                this.time.delayedCall(200, () => {
                    this.player.setFillStyle(0x00ff00);
                });

                if (this.playerHealth <= 0) {
                    this.scene.start('GameOverScene', { score: this.score });
                }
            }

            nextWave() {
                this.wave++;
                this.zombiesKilled = 0;
                this.zombiesPerWave = 5 + (gameData.level * 2) + this.wave;
                
                this.waveText.setText(`Wave: ${this.wave}`);
                
                gameData.coins += 5;
                this.coinsText.setText(`Coins: ${gameData.coins}`);
                
                this.time.delayedCall(2000, () => {
                    this.spawnWave();
                });
            }
        }

        // Upgrade Scene
        class UpgradeScene extends Phaser.Scene {
            constructor() {
                super({ key: 'UpgradeScene' });
            }

            create() {
                this.add.text(187, 40, 'UPGRADES', { 
                    fontSize: '32px', 
                    fill: '#ff0',
                    fontStyle: 'bold'
                }).setOrigin(0.5);

                this.add.text(187, 85, `Coins: ${gameData.coins}`, { 
                    fontSize: '20px', 
                    fill: '#ffd700' 
                }).setOrigin(0.5);

                const upgrades = [
                    { name: 'Damage', key: 'damage', cost: 10, y: 150 },
                    { name: 'Fire Rate', key: 'fireRate', cost: 15, y: 220 },
                    { name: 'Speed', key: 'speed', cost: 12, y: 290 },
                    { name: 'Health', key: 'health', cost: 20, y: 360 }
                ];

                upgrades.forEach(upgrade => {
                    const level = gameData.upgrades[upgrade.key];
                    
                    this.add.text(187, upgrade.y, `${upgrade.name}: Lv ${level}`, { 
                        fontSize: '18px', 
                        fill: '#fff' 
                    }).setOrigin(0.5);

                    const buyButton = this.add.text(187, upgrade.y + 30, `BUY ${upgrade.cost} 🪙`, { 
                        fontSize: '18px', 
                        fill: '#0f0',
                        backgroundColor: '#333',
                        padding: { x: 20, y: 10 }
                    }).setOrigin(0.5).setInteractive();

                    buyButton.on('pointerdown', () => {
                        if (gameData.coins >= upgrade.cost) {
                            gameData.coins -= upgrade.cost;
                            gameData.upgrades[upgrade.key]++;
                            this.scene.restart();
                        }
                    });
                });

                const backButton = this.add.text(187, 500, 'BACK', { 
                    fontSize: '24px', 
                    fill: '#fff',
                    backgroundColor: '#333',
                    padding: { x: 30, y: 12 }
                }).setOrigin(0.5).setInteractive();

                backButton.on('pointerdown', () => {
                    this.scene.start('MenuScene');
                });
            }
        }

        // Game Over Scene
        class GameOverScene extends Phaser.Scene {
            constructor() {
                super({ key: 'GameOverScene' });
            }

            create(data) {
                this.add.text(187, 180, 'GAME OVER', { 
                    fontSize: '42px', 
                    fill: '#f00',
                    fontStyle: 'bold'
                }).setOrigin(0.5);

                this.add.text(187, 250, `Score: ${data.score}`, { 
                    fontSize: '24px', 
                    fill: '#fff' 
                }).setOrigin(0.5);

                this.add.text(187, 290, `Coins: ${gameData.coins}`, { 
                    fontSize: '22px', 
                    fill: '#ffd700' 
                }).setOrigin(0.5);

                const playAgainButton = this.add.text(187, 380, 'PLAY AGAIN', { 
                    fontSize: '24px', 
                    fill: '#0f0',
                    backgroundColor: '#333',
                    padding: { x: 25, y: 12 }
                }).setOrigin(0.5).setInteractive();

                playAgainButton.on('pointerdown', () => {
                    this.scene.start('GameScene');
                });

                const menuButton = this.add.text(187, 450, 'MENU', { 
                    fontSize: '24px', 
                    fill: '#fff',
                    backgroundColor: '#333',
                    padding: { x: 25, y: 12 }
                }).setOrigin(0.5).setInteractive();

                menuButton.on('pointerdown', () => {
                    this.scene.start('MenuScene');
                });
            }
        }
    </script>
</body>
</html>
