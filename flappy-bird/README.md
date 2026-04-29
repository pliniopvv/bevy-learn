# Flappy Bird em Rust com Bevy - Tutorial Técnico

Um clone do clássico Flappy Bird implementado em Rust usando o engine Bevy 0.18. Este tutorial explica detalhadamente as técnicas e algoritmos utilizados no projeto.

## Índice

- [Arquitetura ECS](#arquitetura-ecs)
- [Física e Movimento](#física-e-movimento)
- [Detecção de Colisão](#detecção-de-colisão)
- [Geração Procedural de Canos](#geração-procedural-de-canos)
- [Sistema de Renderização](#sistema-de-renderização)
- [Gerenciamento de Estado](#gerenciamento-de-estado)
- [UI e Pontuação](#ui-e-ponuação)
- [Padrões Avançados do Bevy](#padrões-avançados-do-bevy)

## Arquitetura ECS

O Bevy utiliza a arquitetura **Entity Component System (ECS)**, onde:
- **Entidades** são IDs únicos que agrupam componentes
- **Componentes** são dados puros (structs sem lógica)
- **Sistemas** são funções que processam entidades com componentes específicos

### Definição de Componentes

```rust
#[derive(Component)]
#[require(Gravity(1000.), Velocity)]
struct Player;

#[derive(Component)]
struct Gravity(f32);

#[derive(Component, Default)]
struct Velocity(f32);
```

O atributo `#[require]` garante que toda entidade com `Player` também tenha `Gravity` e `Velocity` automaticamente.

## Física e Movimento

### Simulação de Gravidade

O algoritmo de gravidade aplica aceleração constante na velocidade vertical:

```rust
fn gravity(
    mut transforms: Query<(&mut Transform, &mut Velocity, &Gravity)>,
    time: Res<Time>,
) {
    for (mut transform, mut velocity, gravity) in &mut transforms {
        velocity.0 -= gravity.0 * time.delta_secs();  // Aceleração
        transform.translation.y += velocity.0 * time.delta_secs();  // Integração
    }
}
```

**Fórmula utilizada:**
- `v = v - g × Δt` (atualização da velocidade)
- `y = y + v × Δt` (atualização da posição)

### Controle do Jogador

O pulo é implementado definindo uma velocidade positiva instantânea:

```rust
fn controls(
    mut velocity: Single<&mut Velocity, With<Player>>,
    buttons: Res<ButtonInput<MouseButton>>,
) {
    if buttons.any_just_pressed([MouseButton::Left, MouseButton::Right]) {
        velocity.0 = 400.;  // Impulso vertical para cima
    }
}
```

### Movimento dos Canos

Os canos movem-se horizontalmente a velocidade constante:

```rust
pub const PIPE_SPEED: f32 = 200.0;

fn shift_pipes_to_the_left(
    mut pipes: Query<&mut Transform, With<Pipe>>,
    time: Res<Time>,
) {
    for mut pipe in &mut pipes {
        pipe.translation.x -= PIPE_SPEED * time.delta_secs();
    }
}
```

### Rotação do Pássaro

A rotação é calculada baseada no vetor de velocidade (horizontal = velocidade dos canos, vertical = velocidade do jogador):

```rust
fn enforce_bird_direction(
    mut player: Single<(&mut Transform, &Velocity), With<Player>>,
) {
    let calculated_velocity = Vec2::new(PIPE_SPEED, player.1.0);
    player.0.rotation = Quat::from_rotation_z(calculated_velocity.to_angle());
}
```

## Detecção de Colisão

O Bevy fornece primitivas geométricas através do módulo `bevy::math::bounding`:

### Algoritmo de Colisão

1. **Jogador**: Modelado como `BoundingCircle` (círculo de colisão)
2. **Canos**: Modelados como `Aabb2d` (caixa alinhada aos eixos)
3. **Verificação**: Usa o trait `IntersectsVolume`

```rust
fn check_collisions(
    mut commands: Commands,
    player: Single<(&Sprite, Entity), With<Player>>,
    pipe_segments: Query<(&Sprite, Entity), Or<(With<PipeTop>, With<PipeBottom>)>>,
    pipe_gaps: Query<(&Sprite, Entity), With<PointsGate>>,
    transform_helper: TransformHelper,
) -> Result<()> {
    // Criar collider circular para o jogador
    let player_collider = BoundingCircle::new(
        player_transform.translation().xy(),
        PLAYER_SIZE / 2.,
    );

    // Verificar colisão com cada segmento de cano
    for (sprite, entity) in &pipe_segments {
        let pipe_collider = Aabb2d::new(
            pipe_transform.translation().xy(),
            sprite.custom_size.unwrap() / 2.,
        );

        if player_collider.intersects(&pipe_collider) {
            commands.trigger(EndGame);
        }
    }

    // Verificar passagem pela abertura (pontuação)
    for (sprite, entity) in &pipe_gaps {
        let gap_collider = Aabb2d::new(...);
        if player_collider.intersects(&gap_collider) {
            commands.trigger(ScorePoint);
        }
    }
}
```

**Complexidade**: O(n) onde n é o número de canos ativos.

## Geração Procedural de Canos

### Algoritmo de Posicionamento

A posição vertical da abertura entre canos é determinada por uma **função seno**, criando variação natural:

```rust
fn spawn_pipes(...) {
    let gap_y_position = (time.elapsed_secs() * 4.2309875).sin()
        * CANVAS_SIZE.y / 4.;
    let pipe_offset = PIPE_SIZE.y / 2.0 + GAP_SIZE / 2.0;

    // Cano superior
    Transform::from_xyz(0.0, pipe_offset + gap_y_position, 1.0),
    // Cano inferior
    Transform::from_xyz(0.0, -pipe_offset + gap_y_position, 1.0),
}
```

**Parâmetros:**
- `4.2309875`: Frequência da função seno (controla quão rápido a altura muda)
- `CANVAS_SIZE.y / 4.`: Amplitude (quão alto/baixo os canos podem ir)
- `GAP_SIZE = 100.0`: Tamanho fixo da abertura entre canos

### Spawn Periódico

Usa `on_timer` para spawn a cada 1 segundo:

```rust
spawn_pipes.run_if(on_timer(Duration::from_millis(1000)))
```

## Sistema de Renderização

### Câmera Ortográfica com Resolução Fixa

```rust
Projection::Orthographic(OrthographicProjection {
    scaling_mode: ScalingMode::AutoMax {
        max_width: CANVAS_SIZE.x,   // 480
        max_height: CANVAS_SIZE.y,  // 270
    },
    ..default()
})
```

Isso garante que o jogo tenha dimensões lógicas fixas, independente do tamanho da janela.

### Material2D Customizado com WGSL

Para o background, usamos um material personalizado com shader WGSL:

```rust
#[derive(Asset, TypePath, AsBindGroup, Debug, Clone)]
pub struct BackgroundMaterial {
    #[texture(0)]
    #[sampler(1)]
    pub color_texture: Handle<Image>,
}

impl Material2d for BackgroundMaterial {
    fn fragment_shader() -> ShaderRef {
        "background.wgsl".into()
    }
}
```

### Tiling de Textura

O background usa `ImageAddressMode::Repeat` para repetir a textura:

```rust
asset_server.load_with_settings("background.png", |settings: &mut ImageLoaderSettings| {
    settings.sampler.get_or_init_descriptor()
        .set_address_mode(ImageAddressMode::Repeat);
})
```

### Processamento de Imagem para Pixel Art

Os canos usam `ImageFilterMode::Nearest` para manter bordas nítidas (estilo pixel art):

```rust
settings.sampler.get_or_init_descriptor()
    .set_filter(bevy::image::ImageFilterMode::Nearest);
```

### 9-Slice para Canos

Os canos usam `TextureSlicer` para redimensionamento sem distorcer as bordas:

```rust
SpriteImageMode::Sliced(TextureSlicer {
    border: BorderRect::axes(8., 19.),  // Margens para não distorcer
    center_scale_mode: SliceScaleMode::Stretch,
    ..default()
})
```

## Gerenciamento de Estado

### Events (Eventos)

Eventos são usados para comunicação entre sistemas:

```rust
#[derive(Event)]
struct EndGame;

#[derive(Event)]
pub struct ScorePoint;

// Disparando eventos
commands.trigger(EndGame);
commands.trigger(ScorePoint);
```

### Observers (Observadores)

Observadores reagem a eventos de forma desacoplada:

```rust
// Observer como função
fn respawn_on_endgame(
    _: On<EndGame>,
    mut commands: Commands,
    player: Single<Entity, With<Player>>,
    mut score: ResMut<Score>,
) {
    score.0 = 0;
    commands.entity(*player).insert((
        Transform::from_xyz(-CANVAS_SIZE.x / 4.0, 0.0, 1.0),
        Velocity(0.),
    ));
}

// Observer como closure
.add_observer(
    |_trigger: On<ScorePoint>, mut score: ResMut<Score>| {
        score.0 += 1;
    },
)
```

### Resources (Recursos)

Recursos são dados globais compartilhados:

```rust
#[derive(Resource, Default)]
struct Score(u32);
```

## UI e Pontuação

### UI com Bevy UI

```rust
commands.spawn((
    Node {
        width: percent(100.),
        margin: px(20.).top(),
        ..default()
    },
    Text::new("0"),
    TextLayout::new_with_justify(Justify::Center),
    TextFont { font_size: 33.0, ..default() },
    ScoreText,
));
```

### Atualização Reativa

A pontuação é atualizada apenas quando o recurso muda:

```rust
score_update.run_if(resource_changed::<Score>())
```

## Padrões Avançados do Bevy

### Plugins

Plugins organizam funcionalidades relacionadas:

```rust
pub struct PipePlugin;

impl Plugin for PipePlugin {
    fn build(&self, app: &mut App) {
        app.add_systems(FixedUpdate, (
            despawn_pipes,
            shift_pipes_to_the_left,
            spawn_pipes.run_if(on_timer(Duration::from_millis(1000))),
        ));
    }
}
```

### Hierarquia de Entidades

Canos usam a macro `children![]` para criar entidades filhas:

```rust
commands.spawn((
    Pipe,
    children![
        (PipeTop, Sprite {...}, Transform {...}),
        (PointsGate, Sprite {...}, Transform {...}),
        (PipeBottom, Sprite {...}, Transform {...}),
    ],
));
```

### Query Filters

Filtros poderosos para consultar entidades específicas:

```rust
// Single: garante apenas uma entidade
Single<&mut Velocity, With<Player>>

// Or: múltiplos componentes possíveis
Or<(With<PipeTop>, With<PipeBottom>)>

// Combinações complexas
Single<(&mut Transform, &Velocity), With<Player>>
```

### Encadeamento de Sistemas

`.chain()` garante ordem de execução:

```rust
(
    gravity,
    check_in_bounds,
    check_collisions,
).chain()
```

Isso garante que `gravity` execute antes de `check_in_bounds`, que executa antes de `check_collisions`.

### Despawning de Entidades

Canos fora da tela são removidos para economizar memória:

```rust
fn despawn_pipes(
    mut commands: Commands,
    pipes: Query<(Entity, &Transform), With<Pipe>>,
) {
    for (entity, transform) in pipes.iter() {
        if transform.translation.x < -(CANVAS_SIZE.x / 2.0 + PIPE_SIZE.x) {
            commands.entity(entity).despawn();
        }
    }
}
```

## Conclusão

Este projeto demonstra conceitos fundamentais de desenvolvimento de jogos:
- Física básica com integração numérica
- Detecção de colisão com primitivas geométricas
- Geração procedural com funções trigonométricas
- Padrões ECS para organização de código
- Shaders e materiais customizados
- UI reativa e gerenciamento de estado

O código é um excelente ponto de partida para entender o Bevy e desenvolvimento de jogos em Rust.
