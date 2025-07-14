  <template>
    <div class="home">
      <div class="game-screen">
        <!-- Точки меча -->
        <div
          v-for="(point, index) in swordPoints"
          :key="index"
          class="sword-point"
          :style="{
            left: point.x + 'px',
            bottom: point.y + 'px',
            backgroundColor: point.color,
            display: point.visible ? 'block' : 'none'
          }"
        ></div>

        <!-- Шары -->
        <div
          v-for="(ball, index) in balls"
          :key="'ball-' + index"
          class="ball"
          :style="{
            left: ball.x + 'px',
            bottom: ball.y + 'px',
            backgroundColor: ball.color,
            display: ball.visible ? 'block' : 'none'
          }"
        ></div>

        <!-- Персонажи -->
        <div
          v-for="(player, index) in players"
          :key="index"
          class="character"
          :class="'player' + (index + 1)"
          :style="{
            left: player.bodyLeft + 'px',
            bottom: player.bodyBottom + 'px',
            transform: getCharacterTransform(index),
            filter: player.isRed ? 'hue-rotate(0deg) brightness(1.5) saturate(5)' : 'none',
            backgroundImage: getCharacterSprite(index),
            width: getCharacterWidth(index) + 'px',
            height: getCharacterHeight(index) + 'px'
          }"
        ></div>

        <div class="floor"></div> <!-- Пол игрового поля -->
      </div>
    </div>
  </template>

  <script lang="ts">
  import { Options, Vue } from 'vue-class-component';

  // Импорт спрайтов
  import staySprite from '@/assets/sonya/stay.png';
  import blockSprite from '@/assets/sonya/block.png';
  import preThrowSprite from '@/assets/sonya/pre-throw.png';
  import throwSprite from '@/assets/sonya/thwrow.png';
  import upperAttackSprite from '@/assets/sonya/upper-attack.png';

    // Импорт спрайтов атаки
  import attack1 from '@/assets/main-attack/attack1.png';
  import attack2 from '@/assets/main-attack/attack2.png';
  import attack3 from '@/assets/main-attack/attack3.png';
  import attack4 from '@/assets/main-attack/attack4.png';
  import attack5 from '@/assets/main-attack/attack5.png';
  import attack6 from '@/assets/main-attack/attack6.png';
  import attack7 from '@/assets/main-attack/attack7.png';
  import attack8 from '@/assets/main-attack/attack8.png';
  import attack9 from '@/assets/main-attack/attack9.png';

  // Импорт спрайтов стоячей анимации
  import stay1 from '@/assets/stay/stay1.png';
  import stay2 from '@/assets/stay/stay2.png';
  import stay3 from '@/assets/stay/stay3.png';
  import stay4 from '@/assets/stay/stay4.png';

  // Описание состояния игрока
  interface Player {
    bodyLeft: number;           // X координата тела
    bodyBottom: number;         // Y координата тела (от пола)
    movementSpeed: number;      // Скорость перемещения
    facingLeft: boolean;        // Направление взгляда (влево/вправо)
    movementDirection: -1 | 0 | 1; // Направление движения: влево, стоим, вправо
    isJumping: boolean;         // В прыжке
    isAttacking: boolean;       // Атакует
    isUppercutting: boolean;    // Атака снизу
    isBlocking: boolean;        // Блокирует
    isStunned: boolean;        // Оглушён
    isDead: boolean;           // Убит
    isThrowing: boolean;       // Готовится к броску
    jumpProgress: number;      // Прогресс прыжка
    attackProgress: number;    // Прогресс атаки
    uppercutProgress: number;  // Прогресс атаки снизу
    stunTimer: number;         // Таймер оглушения
    blockProgress: number;     // Прогресс блока
    attackCooldown: number;    // Время перезарядки атаки
    uppercutCooldown: number;  // Время перезарядки атаки снизу
    throwCooldown: number;     // Время перезарядки броска
    throwDelay: number;        // Задержка перед броском
    blockCooldown: number;     // Время перезарядки блока
    isRed: boolean;            // Визуальный эффект при атаке
    lastStayAnimationUpdate: number;
    currentStayFrame: number;            
  }

  // Точка меча
  interface SwordPoint {
    x: number;                 // X координата
    y: number;                 // Y координата
    playerIndex: number;       // Индекс игрока, создавшего точку
    color: string;             // Цвет точки
    visible: boolean;          // Видимость точки
    lifetime: number;          // Время жизни точки
    isUppercut: boolean;       // Это точка от атаки снизу
  }

  // Брошенный шар
  interface Ball {
    x: number;                 // X координата
    y: number;                 // Y координата
    playerIndex: number;       // Индекс игрока, бросившего шар
    color: string;             // Цвет шара
    visible: boolean;          // Видимость шара
    speed: number;             // Скорость шара
    direction: number;         // Направление движения (-1 влево, 1 вправо)
  }

  @Options({})
  export default class HomeView extends Vue {
    // Константы игры
    readonly FLOOR_HEIGHT = 210;
    readonly POINT_LIFETIME = 50;
    readonly POINT_SPACING = 5;
    readonly BALL_SPEED = 10;
    readonly BALL_SIZE = 10;
    readonly JUMP_STRENGTH = 90;
    readonly JUMP_DURATION = 500;
    readonly ATTACK_DURATION = 300;
    readonly UPPERCUT_DURATION = 400; // Длительность атаки снизу
    readonly RED_EFFECT_DURATION = 500;
    readonly ATTACK_COOLDOWN = 1000;
    readonly UPPERCUT_COOLDOWN = 1500; // Длинный кулдаун для атаки снизу
    readonly THROW_COOLDOWN = 1000;
    readonly THROW_DELAY = 300;       // Задержка перед броском
    readonly ACTION_LOCK_DURATION = 300; // Блокировка действий после броска
    readonly BLOCK_DURATION = 500;
    readonly BLOCK_COOLDOWN = 1000;
    readonly STUN_DURATION = 1000;

    // Размеры спрайтов
    readonly SPRITE_WIDTH = 80;
    readonly SPRITE_HEIGHT = 120;
    // cпрайты основной атаки

    readonly ATTACK_SPRITES = [
      attack1, attack2, attack3, attack4, attack5, 
      attack6, attack7, attack8, attack9
    ];
    //спрайты анимации stay
    readonly STAY_SPRITES = [stay1, stay2, stay3, stay4];
    readonly STAY_ANIMATION_DURATION = 1050; // 0.9 секунды на полный цикл
    readonly STAY_ANIMATION_FORWARD_DURATION = 600; // 0.5 секунды на прямую часть
    
    // Статусы игроков
    players: Player[] = [
      {
        bodyLeft: 50, 
        bodyBottom: this.FLOOR_HEIGHT,
        movementSpeed: 5, 
        facingLeft: false, 
        movementDirection: 0,
        isJumping: false, 
        isAttacking: false,
        isUppercutting: false,
        isBlocking: false, 
        isStunned: false,
        isDead: false, 
        isThrowing: false,
        jumpProgress: 0,
        attackProgress: 0, 
        uppercutProgress: 0,
        stunTimer: 0,
        blockProgress: 0, 
        attackCooldown: 0,
        uppercutCooldown: 0,
        throwCooldown: 0, 
        throwDelay: 0,
        blockCooldown: 0,
        isRed: false,
        lastStayAnimationUpdate: 0,
        currentStayFrame: 0,
      },
      {
        bodyLeft: 800, 
        bodyBottom: this.FLOOR_HEIGHT,
        movementSpeed: 5, 
        facingLeft: true, 
        movementDirection: 0,
        isJumping: false, 
        isAttacking: false,
        isUppercutting: false,
        isBlocking: false, 
        isStunned: false,
        isDead: false, 
        isThrowing: false,
        jumpProgress: 0,
        attackProgress: 0, 
        uppercutProgress: 0,
        stunTimer: 0,
        blockProgress: 0, 
        attackCooldown: 0,
        uppercutCooldown: 0,
        throwCooldown: 0, 
        throwDelay: 0,
        blockCooldown: 0,
        isRed: false,
        lastStayAnimationUpdate: 0,
        currentStayFrame: 0,
      }
    ];

    swordPoints: SwordPoint[] = []; // Активные точки мечей
    balls: Ball[] = [];             // Активные шары
    lastTimestamp = 0;
    animationFrameId: number | null = null;

    mounted() {
      window.addEventListener('keydown', this.handleKeyDown);
      window.addEventListener('keyup', this.handleKeyUp);
      this.lastTimestamp = performance.now();
      this.gameLoop();
    }

    beforeUnmount() {
      window.removeEventListener('keydown', this.handleKeyDown);
      window.removeEventListener('keyup', this.handleKeyUp);
      if (this.animationFrameId) cancelAnimationFrame(this.animationFrameId);
    }

    /** === Управление клавишами === **/
    handleKeyDown(event: KeyboardEvent) {
      switch (event.key.toLowerCase()) {
        case 'a': this.startMovement(0, -1); break;
        case 'd': this.startMovement(0, 1); break;
        case 'w': this.initiateJump(0); break;
        case ' ': this.initiateAttack(0); break;
        case 'e': this.initiateUppercut(0); break; // Атака снизу для игрока 1
        case 'q': this.initiateThrow(0); break;
        case 's': this.initateBlock(0); break;
        case 'arrowleft': this.startMovement(1, -1); break;
        case 'arrowright': this.startMovement(1, 1); break;
        case 'arrowup': this.initiateJump(1); break;
        case 'enter': this.initiateAttack(1); break;
        case 'control': this.initiateUppercut(1); break; // Атака снизу для игрока 2
        case 'shift': this.initiateThrow(1); break;
        case 'arrowdown': this.initateBlock(1); break;
      }
    }

    handleKeyUp(event: KeyboardEvent) {
      const key = event.key.toLowerCase();
      if (key === 'a' || key === 'd') this.stopMovement(0);
      if (event.key === 'ArrowLeft' || event.key === 'ArrowRight') this.stopMovement(1);
    }

    // Начало движения
    startMovement(playerIndex: number, direction: -1 | 1) {
      if (!this.canPerformAction(playerIndex)) return;
      this.players[playerIndex].movementDirection = direction;
      this.players[playerIndex].facingLeft = (direction === -1);
    }

    // Остановка движения
    stopMovement(playerIndex: number) {
      this.players[playerIndex].movementDirection = 0;
    }

    // Проверяем — может ли игрок действовать
    canPerformAction(playerIndex: number): boolean {
      const player = this.players[playerIndex];
      return !player.isDead && !player.isStunned && !player.isAttacking && 
            !player.isUppercutting && !player.isBlocking && !player.isThrowing;
    }

    initiateJump(playerIndex: number) {
      const player = this.players[playerIndex];
      if (player.isDead || player.isStunned) return;
      if (player.isJumping) return;
      
      player.isJumping = true;
      player.jumpProgress = 0;
    }

    initiateAttack(playerIndex: number) {
      const player = this.players[playerIndex];
      if (player.isDead || player.isStunned || player.attackCooldown > 0) return;
      if (player.isJumping) {
        // В прыжке можно атаковать без ограничений
        player.isAttacking = true;
        player.attackProgress = 0;
        player.attackCooldown = this.ATTACK_COOLDOWN;
        player.isRed = true;
        setTimeout(() => (player.isRed = false), this.RED_EFFECT_DURATION);
      } else if (!player.isAttacking && !player.isUppercutting && !player.isBlocking && !player.isThrowing) {
        player.isAttacking = true;
        player.attackProgress = 0;
        player.attackCooldown = this.ATTACK_COOLDOWN;
        player.isRed = true;
        setTimeout(() => (player.isRed = false), this.RED_EFFECT_DURATION);
      }
    }

    // Инициирование атаки снизу
    initiateUppercut(playerIndex: number) {
      const player = this.players[playerIndex];
      if (player.isDead || player.isStunned || player.uppercutCooldown > 0) return;
      if (player.isJumping) {
        // В прыжке можно атаковать без ограничений
        player.isUppercutting = true;
        player.uppercutProgress = 0;
        player.uppercutCooldown = this.UPPERCUT_COOLDOWN;
        player.isRed = true;
        setTimeout(() => (player.isRed = false), this.RED_EFFECT_DURATION);
      } else if (!player.isAttacking && !player.isUppercutting && !player.isBlocking && !player.isThrowing) {
        player.isUppercutting = true;
        player.uppercutProgress = 0;
        player.uppercutCooldown = this.UPPERCUT_COOLDOWN;
        player.isRed = true;
        setTimeout(() => (player.isRed = false), this.RED_EFFECT_DURATION);
      }
    }

    initateBlock(playerIndex: number) {
      const player = this.players[playerIndex];
      if (player.isDead || player.isStunned || player.blockCooldown > 0) return;
      if (player.isJumping) {
        // В прыжке можно блокировать без ограничений
        player.isBlocking = true;
        player.blockProgress = 0;
      } else if (!player.isAttacking && !player.isUppercutting && !player.isBlocking && !player.isThrowing) {
        player.isBlocking = true;
        player.blockProgress = 0;
      }
    }

    // Начало броска (с задержкой)
    initiateThrow(playerIndex: number) {
      const player = this.players[playerIndex];
      if (player.isDead || player.isStunned || player.throwCooldown > 0) return;
      if (player.isJumping) {
        // В прыжке можно бросать без задержки
        this.executeThrow(playerIndex);
      } else if (!player.isAttacking && !player.isUppercutting && !player.isBlocking && !player.isThrowing) {
        player.isThrowing = true;
        player.throwDelay = this.THROW_DELAY;
      }
    }

    // Непосредственное выполнение броска
    executeThrow(playerIndex: number) {
      const player = this.players[playerIndex];
      const direction = player.facingLeft ? -1 : 1;
      this.balls.push({
        x: player.bodyLeft + (direction === -1 ? -10 : 30),
        y: player.bodyBottom + 50,
        playerIndex: playerIndex,
        color: playerIndex === 0 ? 'red' : 'purple',
        visible: true,
        speed: this.BALL_SPEED,
        direction
      });
      player.throwCooldown = this.THROW_COOLDOWN;
      player.isThrowing = false;
    }

    /** === Игровой цикл === **/
    gameLoop = (timestamp = 0) => {
      const deltaTime = timestamp - this.lastTimestamp;
      this.lastTimestamp = timestamp;
      this.updateGameState(deltaTime);
      this.animationFrameId = requestAnimationFrame(this.gameLoop);
    };

    updateGameState(deltaTime: number) {
      this.players.forEach((player, index) => {
        // Передвижение
        if (player.movementDirection !== 0 && !player.isDead && !player.isStunned) {
          player.bodyLeft = Math.max(0, Math.min(980, player.bodyLeft + player.movementSpeed * player.movementDirection));
        }

        // Прыжок
        if (player.isJumping) {
          player.jumpProgress += deltaTime;
          const progressFraction = Math.min(player.jumpProgress / this.JUMP_DURATION, 1);
          player.bodyBottom = this.FLOOR_HEIGHT + this.JUMP_STRENGTH * Math.sin(progressFraction * Math.PI);
          if (progressFraction >= 1) {
            player.isJumping = false;
            player.bodyBottom = this.FLOOR_HEIGHT;
          }
        }

        // Обычная атака
        if (player.isAttacking) {
          player.attackProgress += deltaTime;
          this.generateSwordPoints(index, false);
          if (player.attackProgress >= this.ATTACK_DURATION) {
            player.isAttacking = false;
          }
        }

        // Атака снизу
        if (player.isUppercutting) {
          player.uppercutProgress += deltaTime;
          this.generateSwordPoints(index, true);
          if (player.uppercutProgress >= this.UPPERCUT_DURATION) {
            player.isUppercutting = false;
          }
        }

        // Подготовка к броску
        if (player.isThrowing) {
          player.throwDelay -= deltaTime;
          if (player.throwDelay <= 0) {
            this.executeThrow(index);
          }
        }
        // Если игрок мёртв, сбрасываем все активные действия
        if (player.isDead) {
          player.isJumping = false;
          player.isAttacking = false;
          player.isUppercutting = false;
          player.isBlocking = false;
          player.isThrowing = false;
          player.movementDirection = 0;
          return; // Не продолжаем обновлять этого игрока
        }

        // Обновление перезарядок
        player.attackCooldown = Math.max(0, player.attackCooldown - deltaTime);
        player.uppercutCooldown = Math.max(0, player.uppercutCooldown - deltaTime);
        player.throwCooldown = Math.max(0, player.throwCooldown - deltaTime);
        player.blockCooldown = Math.max(0, player.blockCooldown - deltaTime);

        // Блок
        if (player.isBlocking) {
          player.blockProgress += deltaTime;
          if (player.blockProgress >= this.BLOCK_DURATION) {
            player.isBlocking = false;
            player.blockCooldown = this.BLOCK_COOLDOWN;
          }
        }

        // Стан (оглушение)
        if (player.isStunned) {
          player.stunTimer -= deltaTime;
          if (player.stunTimer <= 0) player.isStunned = false;
        }
      });

      // Игроки поворачиваются друг к другу
      if (!this.players[0].isDead && !this.players[1].isDead) {
        this.players[0].facingLeft = this.players[0].bodyLeft + 10 > this.players[1].bodyLeft;
        this.players[1].facingLeft = this.players[1].bodyLeft + 10 > this.players[0].bodyLeft;
      }

      // Обновляем точки меча и шары
      this.updateSwordPoints(deltaTime);
      this.checkCollisions();
      this.updateEnergyBalls();
    }


     // анимации стояния
     getStayAnimationSprite(playerIndex: number): string {
  const player = this.players[playerIndex];
  const now = performance.now();
  
  if (now - player.lastStayAnimationUpdate > 50) {
    const cycleTime = now % this.STAY_ANIMATION_DURATION;
    const frameCount = this.STAY_SPRITES.length;
    
    if (cycleTime < this.STAY_ANIMATION_FORWARD_DURATION) {
      const progress = cycleTime / this.STAY_ANIMATION_FORWARD_DURATION;
      player.currentStayFrame = Math.min(Math.floor(progress * frameCount), frameCount - 1);
    } else {
      const reverseProgress = (cycleTime - this.STAY_ANIMATION_FORWARD_DURATION) / 
                            (this.STAY_ANIMATION_DURATION - this.STAY_ANIMATION_FORWARD_DURATION);
      player.currentStayFrame = frameCount - 2 - Math.floor(reverseProgress * (frameCount - 2));
    }
    
    player.currentStayFrame = Math.max(0, Math.min(player.currentStayFrame, frameCount - 1));
    player.lastStayAnimationUpdate = now;
  }
  
  return `url(${this.STAY_SPRITES[player.currentStayFrame]})`;
}

    // Получение спрайта для персонажа
  getCharacterSprite(playerIndex: number): string {
    const player = this.players[playerIndex];
    
    if (player.isDead) return `url(${this.STAY_SPRITES[0]})`;
    if (player.isStunned) return `url(${this.STAY_SPRITES[0]})`;
    if (player.isAttacking) {
      const frameCount = this.ATTACK_SPRITES.length;
      const progressFraction = Math.min(player.attackProgress / this.ATTACK_DURATION, 1);
      const frameIndex = Math.min(
        Math.floor(progressFraction * frameCount), 
        frameCount - 1
      );
      return `url(${this.ATTACK_SPRITES[frameIndex]})`;
    }
    if (player.isUppercutting) return `url(${upperAttackSprite})`;
    if (player.isBlocking) return `url(${blockSprite})`;
    if (player.isThrowing) {
      return player.throwDelay > 0 ? `url(${preThrowSprite})` : `url(${throwSprite})`;
    }
    
    // Используем новую анимацию стояния
    return this.getStayAnimationSprite(playerIndex);
  }

    // Получение ширины персонажа
    getCharacterWidth(playerIndex: number): number {
      const player = this.players[playerIndex];
      return this.SPRITE_WIDTH;
    }

    // Получение высоты персонажа
    getCharacterHeight(playerIndex: number): number {
      const player = this.players[playerIndex];
      return this.SPRITE_HEIGHT;
    }

    // Трансформация персонажа
    getCharacterTransform(playerIndex: number): string {
      const player = this.players[playerIndex];
      let transform = player.facingLeft ? 'scaleX(-1)' : 'scaleX(1)';
      if (player.isDead) transform += ' rotate(90deg)';
      else if (player.isStunned) transform += ' rotate(10deg)';
      return transform;
    }

    // Создание точек меча при атаке
    generateSwordPoints(playerIndex: number, isUppercut: boolean) {
      const player = this.players[playerIndex];
      let progressFraction, angle;
      
      if (isUppercut) {
        progressFraction = Math.min(player.uppercutProgress / this.UPPERCUT_DURATION, 1);
        angle = (player.facingLeft ? 300 : -300) * progressFraction;
      } else {
        progressFraction = Math.min(player.attackProgress / this.ATTACK_DURATION, 1);
        angle = (player.facingLeft ? -300 : 300) * progressFraction;
      }
      
      const angleRadians = (angle * Math.PI) / 180;
      const baseX = player.bodyLeft + (player.facingLeft ? -10 : this.SPRITE_WIDTH - 10);
      const baseY = player.bodyBottom + this.SPRITE_HEIGHT / 2;
      const pointCount = Math.floor(80 / this.POINT_SPACING);
      
      for (let i = 0; i <= pointCount; i++) {
        const fraction = i / pointCount;
        this.swordPoints.push({
          x: baseX + 80 * fraction * Math.sin(angleRadians),
          y: baseY + 80 * fraction * Math.cos(angleRadians),
          playerIndex: playerIndex,
          color: playerIndex === 0 ? 'red' : 'purple',
          visible: true,
          lifetime: this.POINT_LIFETIME,
          isUppercut: isUppercut
        });
      }
    }

    // Удаление устаревших точек меча
    updateSwordPoints(deltaTime: number) {
      this.swordPoints = this.swordPoints.filter(point => {
        point.lifetime -= deltaTime;
        return point.lifetime > 0 && point.visible;
      });
    }

    // Обновление положений шаров
    updateEnergyBalls() {
    const stepSize = 2; // Длина одного подшага
    const updatedBalls: Ball[] = [];

    for (const ball of this.balls) {
      let steps = Math.ceil(Math.abs(ball.speed * ball.direction) / stepSize);
      let stepDx = (ball.speed * ball.direction) / steps;

      let destroyed = false;

      for (let i = 0; i < steps; i++) {
        ball.x += stepDx;

        // Проверка на выход за пределы
        if (ball.x < -this.BALL_SIZE || ball.x > 1000 + this.BALL_SIZE) {
          destroyed = true;
          break;
        }

        // Проверка столкновения с вражеским шаром
        for (const otherBall of this.balls) {
          if (ball === otherBall) continue;
          if (ball.playerIndex === otherBall.playerIndex) continue;

          const dx = ball.x - otherBall.x;
          const dy = ball.y - otherBall.y;
          const distance = Math.sqrt(dx * dx + dy * dy);

          if (distance < this.BALL_SIZE) {
            destroyed = true;
            otherBall.visible = false;
            break;
          }
        }

        if (destroyed) break;

        // Проверка попадания по игроку
        const targetPlayer = this.players[1 - ball.playerIndex];
        if (!targetPlayer.isDead && this.isColliding(ball.x, ball.y, 1 - ball.playerIndex, this.BALL_SIZE / 2)) {
          this.applyHitEffect(ball.playerIndex, 1 - ball.playerIndex, false);
          destroyed = true;
          break;
        }
      }

      if (!destroyed && ball.visible !== false) {
        updatedBalls.push(ball);
      }
    }

    this.balls = updatedBalls;
  }


    // Проверяем попадания точек и шаров
    checkCollisions() {
    // Попадание мечом по игроку
    this.swordPoints.forEach(point => {
      if (!point.visible) return;
      const targetPlayer = this.players[1 - point.playerIndex];
      if (targetPlayer.isDead) return;

      if (this.isColliding(point.x, point.y, 1 - point.playerIndex, this.POINT_SPACING / 2)) {
        if (point.isUppercut || !targetPlayer.isBlocking) {
          this.applyHitEffect(point.playerIndex, 1 - point.playerIndex, point.isUppercut);
        } else {
          this.stunAttacker(point.playerIndex);
        }
        point.visible = false;
      }
    });

    // Попадание меча по шару
    this.swordPoints.forEach(point => {
      if (!point.visible) return;
      for (let i = 0; i < this.balls.length; i++) {
        const ball = this.balls[i];
        if (ball.playerIndex === point.playerIndex) continue;

        if (this.isPointCollidingWithBall(point, ball)) {
          ball.visible = false;
          point.visible = false;
          break;
        }
      }
    });

    // Попадание шара по игроку (игнорируя блок)
    this.balls = this.balls.filter(ball => {
      const targetPlayer = this.players[1 - ball.playerIndex];
      if (targetPlayer.isDead) return true;

      if (this.isColliding(ball.x, ball.y, 1 - ball.playerIndex, this.BALL_SIZE / 2)) {
        this.applyHitEffect(ball.playerIndex, 1 - ball.playerIndex, false);
        return false;
      }

      return true;
    });

    // Столкновение шаров между собой
    const survivingBalls: Ball[] = [];

    for (let i = 0; i < this.balls.length; i++) {
      const ballA = this.balls[i];
      let destroyed = false;

      for (let j = 0; j < this.balls.length; j++) {
        if (i === j) continue;
        const ballB = this.balls[j];

        // Только вражеские шары могут уничтожать друг друга
        if (ballA.playerIndex === ballB.playerIndex) continue;

        const dx = ballA.x - ballB.x;
        const dy = ballA.y - ballB.y;
        const distance = Math.sqrt(dx * dx + dy * dy);

        if (distance < this.BALL_SIZE) {
          destroyed = true;
          // Удалим также ballB, установив для него флаг
          this.balls[j].visible = false;
          break;
        }
      }

      if (!destroyed && ballA.visible !== false) {
        survivingBalls.push(ballA);
      }
    }

      this.balls = survivingBalls.filter(ball => ball.visible !== false);
    }


    isPointCollidingWithBall(point: SwordPoint, ball: Ball): boolean {
      const dx = point.x - ball.x;
      const dy = point.y - ball.y;
      const distance = Math.sqrt(dx * dx + dy * dy);
      return distance <= this.BALL_SIZE * 1.5;
    }
    // Проверка попадания по телу или голове
    isColliding(x: number, y: number, playerIndex: number, radius: number): boolean {
      const player = this.players[playerIndex];
      const left = player.bodyLeft, bottom = player.bodyBottom;
      const bodyCollision = x >= left - radius && x <= left + this.SPRITE_WIDTH + radius && 
                            y >= bottom - radius && y <= bottom + this.SPRITE_HEIGHT + radius;
      return bodyCollision;
    }

    // Обработка попадания: смерть или стан
    applyHitEffect(attackerIndex: number, targetIndex: number, isUppercut: boolean) {
      const targetPlayer = this.players[targetIndex];
      // Атака снизу пробивает блок
      if (isUppercut || !targetPlayer.isBlocking) {
        targetPlayer.isDead = true;
      } else {
        this.stunAttacker(attackerIndex);
      }
    }

    // Стан атакующего после блока
    stunAttacker(playerIndex: number) {
      const player = this.players[playerIndex];
      player.isStunned = true;
      player.stunTimer = this.STUN_DURATION;
      player.isAttacking = false;
      player.isUppercutting = false;
      player.isBlocking = false;
      player.isThrowing = false;
      player.movementDirection = 0;
    }
  }
  </script>

  <style scoped lang="scss">
  .home {
    display: flex;
    justify-content: center;
    align-items: center;
  }

  .game-screen {
    position: relative;
    width: 1000px;
    height: 600px;
    background: #eef2f3;
    border: 3px solid black;
    overflow: hidden;
  }

  .floor {
    position: absolute;
    bottom: 0;
    left: 0;
    width: 100%;
    height: 35%;
    background: yellow;
    border-top: 3px solid black;
  }

  .character {
    position: absolute;
    background-repeat: no-repeat;
    background-position: center;
    background-size: contain;
    transform-origin: center bottom;
    z-index: 10;
  }

  .player1 {
    filter: hue-rotate(0deg);
  }

  .player2 {
    filter: hue-rotate(120deg);
  }

  .sword-point,
  .ball {
    position: absolute;
    width: 10px;
    height: 10px;
    border-radius: 50%;
    pointer-events: none;
    transform: translate(-50%, 50%);
    z-index: 30;
    box-shadow: 0 0 5px 2px rgba(255, 255, 255, 0.7);
  }

  .ball {
    width: 20px;
    height: 20px;
    z-index: 25;
    box-shadow: 0 0 10px 5px rgba(255, 255, 255, 0.7);
  }
  </style>