
# == Criar cabo de microfone virtual com PipeWire ==
## ==> Para uso com o plugin “Audio Monitor” no OBS Studio

---

#### PRÉ-REQUISITOS
PipeWire como servidor de áudio (padrão na maioria das distros modernas)

#### Testado com:

- SO: BigLinux (Based in Manjaro Linux)
- OBS Studio 32.1.2 ×64 (Flatpak)
- Kernel: 6.6.119
- DE: KDE Plasma 6.6.4
- Plataforma de gráficos: Wayland
- Plugin: “Audio Monitor” 0.10.1, instalado com:
~~~bash
flatpak install com.obsproject.Studio.Plugin.AudioMonitor
~~~

#### PARTE 1 — Criar o microfone virtual

1 - Crie o arquivo de serviço:

~~~bash
cat > ~/.config/systemd/user/mic-virtual.service << 'EOF'
[Unit]
Description=Microfone Virtual PipeWire
After=pipewire.service
Wants=pipewire.service

[Service]
ExecStart=pw-loopback \
–capture-props='media.class=Audio/Sink node.name=mic-virtual node.description="Microfone Virtual"' \
–playback-props='media.class=Audio/Source node.name=mic-virtual-source node.description="Microfone Virtual Source"'
Restart=on-failure

[Install]
WantedBy=default.target
EOF
~~~

2 - Ative e inicie o serviço:

~~~bash
systemctl –user daemon-reload
~~~

~~~bash
systemctl –user enable –now mic-virtual.service
~~~

3 - Verifique se os dispositivos foram criados:

~~~bash
pactl list sources short | grep mic-virtual
~~~

\# Deve aparecer algo como: <br>
\# mic-virtual-source PipeWire float32le 2ch 48000hz RUNNING <br>
\# Se não aparecer nada, verifique com:

~~~bash
systemctl –user status mic-virtual
~~~

#### PARTE 2 — Direcionar o Audio Monitor do OBS para o mic virtual

Por padrão, o plugin “Audio Monitor” do OBS envia o áudio para o alto-falante/fone. Esta regra força o envio para o mic virtual.

4 - Crie a pasta de configuração (se não existir) e o arquivo de regras:

~~~bash
mkdir -p ~/.config/pipewire/pipewire.conf.d
~~~

~~~bash
cat > ~/.config/pipewire/pipewire.conf.d/obs-monitor-link.conf << 'EOF'
context.modules = [
    {
        name = libpipewire-module-link-factory
    }
]
context.rules = [
    {
        matches = [{ node.name = "OBS-Monitor" }]
        actions = {
            create-links = {
                audio.position = [ FL FR ]
                node.target = "mic-virtual"
            }
        }
    }
]
EOF
~~~

5 - Reinicie o PipeWire e o serviço do Mic Virtual, para aplicar as regras:

~~~bash
systemctl –user restart pipewire pipewire-pulse
~~~

~~~bash
systemctl –user restart mic-virtual
~~~

### PARTE 3 — Configurar o OBS e o Google Meet

No OBS:

- Instale o plugin “Audio Monitor” se ainda não tiver
- No menu de painéis, adicione o painel “Monitor de Audio” e encaixe na interface
- No botão de configurações do plugin, vá em “Saídas” -> “Faixa 1”
- Selecione o “Microfone Virtual”

No Google Meet (ou qualquer app de videoconferência):

- Microfone → selecione “Microfone Virtual Source”

#### OBSERVAÇÕES

- O serviço sobe automaticamente com o sistema a partir de agora
- Para monitorar o que está sendo enviado, use o monitor nativo do
  OBS (isso não interfere no que o Meet recebe)
- Para ver o roteamento de áudio visualmente: instale o qpwgraph ou o Helvum
