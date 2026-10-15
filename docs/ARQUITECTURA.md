# Arquitectura

Lo que hace Proyecto Musical es separar **entrada (cámara)**, **entendimiento (IA/gesto)**,
**salida (audio)** y **presentación (pygame)** en componentes desacoplados.
Esto permite testear casi todo sin cámara ni dispositivo de audio.

---

## Diagrama de componentes

```
┌────────────────────────────────────────────────────────────────────────────┐
│                              HILO PRINCIPAL (pygame)                         │
│                                                                              │
│   pygame events ──► ManoMusicalApp._handle_events                            │
│                         │                                                    │
│   SharedState.drain_events ──► _trigger_on/_trigger_off ──► AudioEngine      │
│                         │                              └──► Renderer        │
│   JingleBellsGame.update / SongPlayer.update / PerformancePlayer.update      │
│                         │                                                    │
│   Renderer.draw_* ──► pygame.display.flip()                                  │
└───────────────▲───────────────────────────────────────────┬─────────────────┘
                │ get_frame / get_hand_frame                 │ set_frame
                │ push_event                                 │ set_hand_frame
┌───────────────┴───────────────┐              ┌─────────────▼─────────────────┐
│      InferenceThread          │              │        CameraThread           │
│  HandTracker (MediaPipe)      │              │  cv2.VideoCapture + flip      │
│  FingerPressDetector          │              │  (publica el último frame)     │
│  ──► NoteTrigger ──► queue ───┘              └───────────────────────────────┘
└───────────────────────────────┘
```

- El **render** nunca espera a la IA: siempre dibuja el último frame/estado disponible.
- El hilo de inferencia **descarta frames viejos** comparando `frame_id`; si la IA
  va más lenta que la cámara, se salta frames en lugar de acumular latencia.

---

## Módulos

| Módulo | Responsabilidad | Depende de |
|--------|-----------------|------------|
| `config.py` | Constantes, rutas, mapa de notas, paleta | — |
| `music.py` | Teoría (nota↔MIDI↔freq), síntesis, `Song` | numpy, config |
| `audio_engine.py` | Polifonía sobre `pygame.mixer`, caché de samples | pygame, music |
| `hand_tracker.py` | Wrapper `HandLandmarker` (MediaPipe Tasks) | mediapipe, cv2 |
| `gesture.py` | Ángulos de dedo, histéresis, triggers, swipe | numpy, hand_tracker |
| `state.py` | `SharedState` seguro entre hilos | hand_tracker |
| `capture.py` | `CameraThread` + `InferenceThread` | cv2, gesture, hand_tracker |
| `render.py` | Cámara, esqueleto, teclas, partículas, HUD | pygame, config |
| `game.py` | `JingleBellsGame`, `SongPlayer` | music, config |
| `recorder.py` | Grabar/reproducir performances | config, json |
| `app.py` | Orquestador: bucle, modos, teclas | todos |
| `utils.py` | Logging, descarga de modelo, helpers | config |


### Flujo de datos de una nota (happy path)

```
frame BGR
  └─ CameraThread.set_frame(frame, ts)
        └─ InferenceThread: HandTracker.process(frame, ts) -> HandFrame(21×3 landmarks)
              └─ FingerPressDetector.update(HandFrame, now) -> [NoteTrigger(on, B4, vel=0.8)]
                    └─ SharedState.push_event(trigger)
                          └─ (hilo principal) drain_events -> AudioEngine.note_on("Right:index","B4",0.8)
                                                              -> Renderer.burst_note("B4", ...)
                                                              -> JingleBellsGame.register_input("B4", now)
```

---

## Pruebas sin hardware

- `tests/test_gesture.py` construye landmarks sintéticos: no necesita cámara.
- `tests/test_recorder.py` verifica `AudioEngine(enabled=False)`.
- Smoke test headless con `SDL_VIDEODRIVER=dummy` y `SDL_AUDIODRIVER=dummy`.
