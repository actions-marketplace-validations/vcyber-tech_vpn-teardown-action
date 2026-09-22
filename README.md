Encerra o túnel VPN aberto por [vpn-setup-action](https://github.com/vcyber-tech/vpn-setup-action)
e limpa os arquivos de trabalho. **Idempotente** — pode rodar várias vezes sem
efeito colateral, e não falha se não houver nada para derrubar.

Esta action faz parte do monorepo [**actions-rust**](https://github.com/vcyber-tech/actions-rust).
O trabalho pesado é feito pelo `vpnctl`, um binário Rust estaticamente
linkado (musl), distribuído pelas releases do monorepo.

## Por que usar esta action

Túneis VPN abertos em runners precisam ser fechados de forma limpa:

- **Em runners self-hosted**, a interface `tun0` persiste entre jobs. Sem
  teardown, o próximo job herda estado sujo e o próximo `openvpn` falha.
- **No servidor de VPN**, a sessão do cliente só expira após o DPD (Dead
  Peer Detection), o que leva de 30s a 2min. Um teardown explícito libera o
  IP do pool imediatamente.

Esta action faz as duas coisas: envia SIGTERM ao `openvpn`, aguarda o
processo encerrar, e remove os arquivos de trabalho.

## Uso

Combine com o `vpn-setup-action`, sempre com `if: always()` para garantir
que rode mesmo se um step anterior falhar:

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: VPN Setup
        id: vpn
        uses: vcyber-tech/vpn-setup-action@v1
        with:
          config: ${{ secrets.VPN_CONFIG_INLINE }}
          healthcheck-host: internal.dns.example
          healthcheck-port: '53'

      - name: Deploy
        run: |
          # ... comandos que dependem do túnel ...

      # Teardown SEMPRE com `if: always()`.
      - name: VPN Teardown
        if: always()
        uses: vcyber-tech/vpn-teardown-action@v1
```

**Por que** if: always(): sem ele, se o deploy falhar antes do teardown,
a VPN fica de pé — exatamente o cenário que a action existe para evitar.

## Inputs

| Input | Obrigatório | Padrão | Descrição |
|---|---|---|
| `version` | não | -- | Versão do `vpnctl` no monorepo (ex: `vpn-teardown/v3.2.0`). Se vazio, é derivada da tag da action. |

## Comportamento em falha

Esta action é best-effort por design:

- Se o vpnctl não estiver disponível, emite warning e sai com sucesso

- Se o vpnctl desconectar retornar erro, emite warning e sai com sucesso

- Se não houver PID file (nada para derrubar), reporta nenhum openvpn em execução e sai com sucesso

Isso é intencional: teardown nunca deve mascarar um erro real do job com
um erro próprio. Se o deploy falhou, você quer ver **o erro do deploy**,
não um erro secundário do teardown.

## Cache compartilhado com o vpn-setup

Se `vpn-setup-action` já rodou no mesmo job, o binário `vpnctl` está em
`$RUNNER_TEMP/toolshed-vpnctl` e o teardown o reaproveita — sem baixar de
novo. **Isso é automático, não requer configuração.**

## Requisitos

- Linux runner (x86_64)

- `sudo` com NOPASSWD (padrão em runners `ubuntu-latest` do GitHub; configure explicitamente em runners self-hosted)

## Código-fonte e issues

- Monorepo (código-fonte): [**vcyber-tech/actions-rust**](https://github.com/vcyber-tech/actions-rust)

- Issues: [abrir uma issue](https://github.com/vcyber-tech/actions-rust/issues)

- Changelog: [releases](https://github.com/vcyber-tech/actions-rust/releases)

## Licença

MIT — veja [LICENSE](./LICENSE).
