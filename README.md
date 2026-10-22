# pixel-sim

Pixel Display Simulator for the Divoom **Pixoo-64** and **Pixoo-64 II**.

pixel-sim accepts the same local HTTP API as the real devices (`POST /post`, and `/divoom_api` on newer firmware) and renders the 64×64 display live in a web page. You can check that your animations, text and layouts look and behave as intended before you push them to a real device. To use it, point your existing client or library at the simulator's address instead of the device's IP. The simulator also warns about requests that would misbehave on real hardware.

> Status: design phase. See the [Architecture Decision Records](docs/adr/README.md) for the idea, the scope and the architecture.
