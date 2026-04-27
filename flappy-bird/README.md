# Flappy Bird - Rust + Bevy

Implementação do clássico Flappy Bird usando Rust e o engine Bevy 0.18.

## Tutorial

https://www.rustadventure.dev/flappy-bird/bevy-0.18/how-to-start-a-new-bevy-game

## Técnicas Utilizadas

### Engine e Arquitetura
- **Bevy 0.18.1** - Engine de jogos data-driven em Rust
- **ECS (Entity Component System)** - Arquitetura core do Bevy com separação clara entre entidades, componentes e sistemas

### Componentes
- `Player`, `Pipe`, `PipeTop`, `PipeBottom` - Marcadores de entidades
- `Gravity(f32)` - Aceleração vertical aplicada ao jogador
- `Velocity(f32)` - Velocidade vertical do jogador
- `PointsGate` - Área de colisão para pontuação
- `ScoreText` - Marcador para UI de pontuação

### Resources e Events
- `Score(u32)` - Pontuação atual do jogo
- `EndGame` - Evento disparado em colisões/mortes
- `ScorePoint` - Evento disparado ao passar por um cano

### Sistemas e Schedules
- **FixedUpdate**: `gravity`, `check_in_bounds`, `check_collisions`, `despawn_pipes`, `shift_pipes_to_the_left`, `spawn_pipes`
- **Update**: `controls`, `score_update`, `enforce_bird_direction`
- **Startup**: `startup` - Inicialização de câmera, jogador, UI e background
- Encadeamento de sistemas com `.chain()` para garantir ordem de execução

### Física e Movimento
- Gravidade simulada com aceleração constante (1000 u/s²)
- Movimento baseado em velocidade integrada no tempo (`delta_secs`)
- Canos deslocam-se horizontalmente a velocidade constante (200 u/s)
- Rotação do pássaro baseada na direção do movimento (velocidade + pipe speed)

### Detecção de Colisão
- `BoundingCircle` para o jogador
- `Aabb2d` para canos e portões de pontuação
- Trait `IntersectsVolume` para verificação de colisões
- `TransformHelper` para computar transformações globais de hierarquias

### Renderização 2D
- **Sprites** com `Sprite` component e `custom_size`
- **Material2D customizado** com WGSL shader (`background.wgsl`)
- `Material2dPlugin<BackgroundMaterial>` para materiais personalizados
- `AsBindGroup` para bind groups automáticos de texturas
- `Mesh2d` + `MeshMaterial2d` para renderização de malha de fundo
- Câmera ortográfica com `ScalingMode::AutoMax` para resolução fixa (480x270)

### Processamento de Imagem
- `ImageLoaderSettings` para configuração de texturas
- `ImageAddressMode::Repeat` para textura de fundo em tiling
- `ImageFilterMode::Nearest` para canos (pixel art)
- `TextureSlicer` com `SliceScaleMode::Stretch` para redimensionamento de canos

### Geração Procedural
- Posicionamento vertical dos canos usando função seno: `(time * 4.23).sin() * (height/4)`
- Gap (abertura) de tamanho fixo (100 u) entre canos superior e inferior

### Padrões Bevy
- **Plugins**: `PipePlugin` encapsula lógica de canos
- **Observers**: `respawn_on_endgame` e closure para `ScorePoint`
- **Query filters**: `With<T>`, `Single<&T, With<U>>`, `Or<(With<A>, With<B>)>`
- **Run conditions**: `on_timer(Duration)`, `resource_changed::<Score>()`
- **Hierarquia de entidades**: canos usam `children![]` macro
- `require` attribute para componentes obrigatórios (Player requer Gravity e Velocity)

### UI
- UI do Bevy com `Node` (100% width) e `Text`
- `TextLayout::new_with_justify(Justify::Center)` para centralização
- Atualização reativa da pontuação via `resource_changed`
