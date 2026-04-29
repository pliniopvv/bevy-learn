# Block Breaker - Tutorial de Bevy 2D

Um jogo estilo Breakout/Arkanoid feito com Bevy 0.18, demonstrando várias técnicas de desenvolvimento de jogos 2D com Entity Component System (ECS).

## 🎮 Sobre o Jogo

O jogador controla uma plataforma (paddle) para rebater uma bola e quebrar tijolos. O jogo termina quando a bola cai na área inferior de respawn. Pressione 'R' para reiniciar.

## 📚 Técnicas e Conceitos do Bevy

### 1. Entity Component System (ECS)

O Bevy utiliza arquitetura ECS, onde:
- **Entities**: IDs únicos que representam objetos do jogo
- **Components**: Dados anexados às entities (ex: `Transform`, `Velocity`, `Ball`)
- **Systems**: Funções que processam entities com components específicos
- **Resources**: Dados globais acessíveis por systems (ex: `Time`, `AppState`)

### 2. Components Utilizados

```rust
// Marker Components (identificadores)
#[derive(Component)]
struct Ball;      // Identifica a bola
#[derive(Component)]
struct Paddle;    // Identifica a plataforma
#[derive(Component)]
struct Brick;     // Identifica os tijolos
#[derive(Component)]
struct Wall(Plane2d);  // Paredes com orientação de plano
#[derive(Component)]
struct RespawnBallArea; // Área de respawn

// Data Components
#[derive(Component)]
struct Velocity(Vec2);    // Velocidade da bola
#[derive(Component)]
struct HalfSize(Vec2);    // Meio-tamanho para colisão AABB
```

### 3. Gerenciamento de Estado (State Pattern)

O jogo utiliza o sistema de estados do Bevy para controlar o fluxo:

```rust
#[derive(States, Debug, Clone, Copy, Eq, PartialEq, Hash, Default)]
enum AppState {
    #[default]
    GameOver,  // Tela inicial/fim de jogo
    Playing,   // Jogo em execução
}
```

**Transições de estado:**
- `GameOver` → `Playing`: Quando 'R' é pressionado
- `Playing` → `GameOver`: Quando a bola cai na área de respawn

**Entidades atreladas ao estado:**
```rust
// Entidades criadas no estado Playing são removidas ao sair
commands.spawn((
    Ball,
    DespawnOnExit(AppState::Playing),  // Auto-limpeza!
    // ...
));
```

### 4. Sistemas e Agendamento (Schedules)

```rust
app.add_systems(Startup, startup)
   .add_systems(OnEnter(AppState::Playing), new_game)
   .add_systems(OnEnter(AppState::GameOver), show_restart_button)
   .add_systems(Update, restart_game.run_if(...))
   .add_systems(FixedUpdate, (
       paddle_controls,
       ball_movement,
       on_intersect_respawn_area,
   ));
```

- **Startup**: Executa uma vez no início
- **OnEnter/OnExit**: Executa ao entrar/sair de um estado
- **Update**: Executa a cada frame
- **FixedUpdate**: Executa em intervalos fixos (física consistente)

---

## 🧮 Algoritmos Implementados

### 1. Movimento da Bola (Frame-rate Independent)

```rust
fn ball_movement(time: Res<Time>, mut query: Query<(&mut Transform, &mut Velocity, &Ball)>) {
    let ball_move_distance = velocity.0.length() * time.delta_secs();
    let ball_ray = Ray2d::new(transform.translation.xy(), velocity.0.normalize());

    transform.translation += (velocity.0 * time.delta_secs()).extend(0.);
}
```

O movimento é multiplicado pelo `delta_secs()` para garantir velocidade consistente independente do FPS.

### 2. Detecção de Colisão (3 Camadas)

O sistema de colisão é sofisticado e usa três níveis de verificação:

#### **Camada 1: Colisão com Paredes (Ray-Plane Intersection)**

```rust
for (wall, origin) in walls {
    if let Some(hit_distance) = ball_ray.intersect_plane(origin.translation.xy(), wall.0)
        && hit_distance <= ball_move_distance {
            velocity.0 = velocity.0.reflect(wall.0.normal.as_vec2());
            return;  // Colisão com parede tem prioridade
    }
}
```

Utiliza interseção de raio com plano 2D. Se o raio da bola intersecta uma parede dentro da distância percorrida, a velocidade é refletida usando o vetor normal da parede.

#### **Camada 2: Colisão com AABB (Ray-AABB Intersection)**

Para tijolos e plataforma (caixas alinhadas aos eixos):

```rust
let ball_cast = RayCast2d::from_ray(ball_ray, ball_move_distance);

let collisions: Vec<_> = aabb_colliders
    .iter()
    .filter_map(|(entity, origin, half_size)| {
        let aabb = Aabb2d::new(origin.translation.xy(), half_size.0);
        let hit_distance = ball_cast.aabb_intersection_at(&aabb)?;
        Some((entity, origin, aabb, hit_distance))
    })
    .collect();
```

Cria um `RayCast2d` e testa interseção com cada AABB. Usa `filter_map` para extrair apenas colisões válidas.

#### **Camada 3: Seleção da Colisão Mais Próxima**

```rust
let (entity, origin, aabb, hit_distance) = collisions
    .into_iter()
    .min_by_key(|(_, _, _, distance)| FloatOrd(*distance))
    .unwrap();
```

Quando múltiplos objetos podem ser atingidos, apenas o mais próximo é processado (usando `FloatOrd` para comparar floats).

### 3. Cálculo da Normal de Colisão no AABB

Para determinar qual face do AABB foi atingida:

```rust
let (hit_normal, _) = [
    (Plane2d::new(Vec2::NEG_Y), Vec2::new(origin.translation.x, aabb.min.y)),  // Face inferior
    (Plane2d::new(Vec2::Y), Vec2::new(origin.translation.x, aabb.max.y)),      // Face superior
    (Plane2d::new(Vec2::NEG_X), Vec2::new(aabb.min.x, origin.translation.y)),  // Face esquerda
    (Plane2d::new(Vec2::X), Vec2::new(aabb.max.x, origin.translation.y)),      // Face direita
].into_iter()
    .filter_map(|(plane, location)| {
        ball_ray.intersect_plane(location, plane)
            .map(|hit_distance| (plane.normal, hit_distance))
    })
    .min_by_key(|(_, distance)| FloatOrd(*distance))
    .unwrap();

velocity.0 = velocity.0.reflect(hit_normal);
```

Testa interseção com as 4 faces do AABB e escolhe a face mais próxima para calcular o vetor normal de reflexão.

### 4. Reflexão na Plataforma (Paddle) - Ângulo Variável

Diferente das paredes, a plataforma usa um algoritmo de ângulo baseado na posição de impacto:

```rust
if paddles.get(entity).is_ok() {
    let direction_vector = transform.translation.xy() - origin.translation.xy();
    let angle = direction_vector.to_angle();
    let linear_angle = angle.clamp(0., PI) / PI;  // Normaliza para 0-1
    let softened_angle = FRAC_PI_4.lerp(PI - FRAC_PI_4, linear_angle);  // Suaviza para 45°-135°
    velocity.0 = Vec2::from_angle(softened_angle) * velocity.0.length();
}
```

**Como funciona:**
- Bola atinge centro da plataforma → 90° (rebate reto para cima)
- Bola atinge borda esquerda → 135° (45° da vertical para esquerda)
- Bola atinge borda direita → 45° (45° da vertical para direita)

Isso cria um gameplay mais dinâmico e controlável.

### 5. Movimento da Plataforma

```rust
if input.pressed(KeyCode::KeyA) {
    transform.translation.x -= PADDLE_SPEED * time.delta_secs();
} else if input.pressed(KeyCode::KeyD) {
    transform.translation.x += PADDLE_SPEED * time.delta_secs();
}
```

Movimento simples com teclas A/D, também independente do frame rate.

### 6. Detecção de Queda (Respawn Area)

```rust
let ball_collider = BoundingCircle::new(ball.translation.xy(), BALL_SIZE);
let respawn_collider = Aabb2d::new(respawn_area.0.translation.xy(), respawn_area.1.custom_size.unwrap() / 2.);

if ball_collider.intersects(&respawn_collider) {
    next_state.set(AppState::GameOver);
}
```

Usa interseção entre `BoundingCircle` (bola) e `Aabb2d` (área de respawn).

### 7. Layout dos Tijolos

```rust
let num_bricks_per_row = 13;
let rows = 6;

for row in 0..rows {
    for i in 0..num_bricks_per_row {
        let x = BRICK_SIZE.x * i as f32
              - BRICK_SIZE.x * num_bricks_per_row as f32 / 2.
              + BRICK_SIZE.x / 2.;
        let y = CANVAS_SIZE.y * (3./8.) - BRICK_SIZE.y * row as f32;
        // spawn brick at (x, y)
    }
}
```

Cria uma grade de 13×6 = 78 tijolos, centralizada horizontalmente e posicionada no terço superior da tela.

---

## 🎨 Técnicas de Renderização

### Câmera Ortográfica com Auto-Scaling

```rust
Projection::Orthographic(OrthographicProjection {
    scaling_mode: ScalingMode::AutoMin {
        min_width: CANVAS_SIZE.x + BRICK_SIZE.x,
        min_height: CANVAS_SIZE.y + BRICK_SIZE.y,
    },
    ..default()
})
```

Garante que a viewport mínima seja sempre visível, independente do tamanho da janela.

### Ordenação Z (Layering)

- Z = -3: Fundo (Sky 50)
- Z = -2: Área de jogo (Sky 800)
- Z = -1: Área de respawn (Sky 500 com alpha)
- Z = 0: Bola, plataforma, tijolos
- Z = +1: Detalhes internos (círculo da bola, interior dos tijolos)

### Cores com OKLCH e Paleta Tailwind

```rust
let base_color = Oklcha::from(SKY_400);
let current_color = base_color.with_hue(((row + i) % 8) as f32 * (num_bricks_per_row * rows) as f32);
```

Usa espaço de cor OKLCH para gerar um efeito arco-íris nos tijolos, variando apenas o matiz (hue).

### Entidades Filhas (Child Entities)

```rust
commands.spawn((
    Ball,
    // ...
)).with_children(|parent| {
    parent.spawn((
        Mesh2d(meshes.add(Circle::new(BALL_SIZE / 2.))),
        MeshMaterial2d(materials.add(ColorMaterial::from(Color::srgb(1., 1., 1.)))),
        Transform::from_xyz(0., 0., 1.),  // Z +1 para ficar na frente
    ));
});
```

Cria um círculo menor branco dentro da bola, e um retângulo menor dentro de cada tijolo (efeito de borda).

---

## 📁 Estrutura do Código

```
src/
  main.rs (399 linhas)
  ├── Imports e definições
  ├── Components e Resources
  ├── Setup (startup, new_game, show_restart_button)
  ├── Input systems (paddle_controls, restart_game)
  ├── Physics (ball_movement com colisão)
  └── Detection (on_intersect_respawn_area)
```

O projeto é um arquivo único, ideal para aprendizado e tutoriais.

---

## 🎯 Constantes do Jogo

| Constante | Valor | Descrição |
|-----------|-------|-----------|
| `BALL_SIZE` | 10.0 | Raio da bola |
| `BRICK_SIZE` | (80, 40) | Dimensões do tijolo |
| `CANVAS_SIZE` | (1280, 720) | Tamanho da área de jogo |
| `DEFAULT_PADDLE_SIZE` | (200, 20) | Dimensões da plataforma |
| `PADDLE_SPEED` | 400.0 | Velocidade em pixels/segundo |

---

## 🚀 Executando

```bash
cargo run
```

**Controles:**
- A / D: Mover plataforma esquerda/direita
- R: Reiniciar (após game over)

---

## 📖 Conceitos Demonstrados

1. **ECS Pattern**: Components, queries, commands
2. **State Management**: Estados do jogo com transições
3. **FixedUpdate**: Física determinística
4. **Ray Casting**: Interseção raio-plano e raio-AABB
5. **Collision Response**: Reflexão vetorial com normais
6. **Variable Bounce**: Ângulo baseado em posição relativa
7. **DespawnOnExit**: Limpeza automática de entidades
8. **Color Math**: OKLCH color space manipulation
9. **Child Entities**: Hierarquia de transformações
10. **Run Conditions**: `run_if()` para filtrar execução de sistemas
