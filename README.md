# Crossref Reference Linking — temporary OJS 3.5 hot-fix build

[![OJS](https://img.shields.io/badge/OJS-3.5-brightgreen)](https://pkp.sfu.ca/ojs/)
[![Version](https://img.shields.io/badge/version-3.0.0.90-blue)](version.xml)
[![License](https://img.shields.io/badge/license-GPL--3.0-lightgrey)](LICENSE)

**⬇️ Install package:** [OJS 3.5](https://github.com/OJSBR/crossrefReferenceLinking/releases/download/3.0.0.90/crossrefReferenceLinking-3.0.0.90.tar.gz) — or browse all [Releases](../../releases).

> [!IMPORTANT]
> **This is not an OJSBR plugin and it is not meant to live long.**
>
> It is PKP's own `crossrefReferenceLinking` plugin, release 3.0.0.2, carrying **one single
> change**: the fix for [pkp/crossrefReferenceLinking#52](https://github.com/pkp/crossrefReferenceLinking/issues/52)
> — the scheduled task that retrieves the reference DOIs is never registered when the scheduler
> runs from the command line, so on installations driven by cron the DOIs of the references are
> never fetched.
>
> The fix is already proposed upstream in
> [pkp/crossrefReferenceLinking#53](https://github.com/pkp/crossrefReferenceLinking/pull/53).
> **As soon as PKP merges it and releases the plugin, this build is discontinued** — this
> repository will be archived and every installation should go back to the plugin from the
> PKP Plugin Gallery. Nothing here is a feature of ours, and no feature request belongs here:
> it all belongs upstream.

## The bug, in one paragraph

`CrossrefReferenceLinkingPlugin::registerSchedules()` asks whether the plugin is enabled, whether
the Crossref credentials are set and whether citations metadata is on — **without passing a
context id**. `PKPScheduler::registerPluginSchedules()` runs under the CLI, where there is no
request and therefore no current journal, so the three checks resolve against no context, return
false, and `CrossrefReferenceLinkingInfoSender` is never registered. Articles are deposited, keep
`crossref::checkCitationsDOIs = 1` forever, and their citations never receive the DOIs Crossref
has already matched for them.

With `[schedule] task_runner = On` the task is registered by the web based runner instead, using
the context of whichever request happened to trigger it — so the problem looks intermittent rather
than broken, and tends to hide itself on busy multi-journal installations.

The guards are also redundant: `CrossrefReferenceLinkingInfoSender::getJournals()` already filters
journal by journal with the explicit id, which is the only place a per-journal setting can be
evaluated meaningfully.

## The whole difference from upstream

```diff
     public function registerSchedules(PKPScheduler $scheduler): void
     {
-        if (!$this->getEnabled() ||
-            !$this->hasCrossrefCredentials() ||
-            !$this->citationsEnabled()) {
-                return;
-        }
+        // No context is available when the scheduler registers the plugin schedules,
+        // so the plugin configuration cannot be consulted here. The task itself filters
+        // the journals it has to process, in CrossrefReferenceLinkingInfoSender::getJournals().
         $scheduler
             ->addSchedule(new CrossrefReferenceLinkingInfoSender([]))
             ->hourly()
```

Plus the release number in `version.xml` and this file. Nothing else.

## Do you even need this?

Only if the plugin is enabled in at least one journal. Then, in the OJS directory:

```bash
php lib/pkp/tools/scheduler.php list | grep -i crossrefReferenceLinking
```

Silence means the task will never run from cron on that installation — you are affected. How much
it has cost you so far:

```sql
SELECT count(*) FROM submission_settings
 WHERE setting_name = 'crossref::checkCitationsDOIs' AND setting_value = '1';
```

Those are deposited articles waiting for reference DOIs that nobody is going to fetch.

To catch up on the backlog after installing this build, the plugin's own command line tool does it
without waiting for the next hour:

```bash
php plugins/generic/crossrefReferenceLinking/tools/checkCrossrefReferencesDOIs.php all
```

## Why the release number is 3.0.0.90

`PluginHelper::upgradePlugin()` refuses any package that is not strictly newer than the installed
version, so a build numbered 3.0.0.2 could not be installed over 3.0.0.2 at all. The jump to
**3.0.0.90** also buys protection: the Plugin Gallery compares the `versions` table against PKP's
feed and reports `PLUGIN_GALLERY_STATE_NEWER` — *a newer version is installed* — so it will not
offer an upgrade, and nobody overwrites the fix with a single click.

| situation | what the Plugin Gallery shows |
|---|---|
| this build (3.0.0.90), PKP at 3.0.0.2 | newer version installed — no upgrade offered |
| PKP releases 3.0.0.x without the fix | still newer version installed — the fix stays |
| PKP releases 3.0.1.0 with the fix | upgrade offered again — **go back to the standard plugin** |

That last row is the exit signal. Release strings are read as four integers, which is why the
number could not be something like `3.0.0.2-ojsbr1`.

## Installation

Site Administrator → **Settings → Website → Plugins**, find *Crossref Reference Linking* in the
installed plugins list, choose **Upgrade** and upload the `.tar.gz` from
[Releases](../../releases). OJS validates the name and category, replaces the plugin directory and
records the new version.

Nothing else changes: the directory name, the plugin identity and the class are the same as
upstream, so the Crossref credentials, the journal settings and every `crossref::*` value already
stored on your submissions and citations are preserved.

## Going back to the standard plugin

When PKP releases the plugin with the fix, the Plugin Gallery will offer the upgrade again — take
it. That is all; there is nothing of ours to uninstall.

## Credits

The plugin is the work of the **[Public Knowledge Project](https://pkp.sfu.ca)** and its
contributors, distributed under the GNU GPL v3. Upstream repository:
[pkp/crossrefReferenceLinking](https://github.com/pkp/crossrefReferenceLinking). The upstream
documentation is kept in [`README`](README).

This build is published by **[OJSBR](https://ojsbr.com.br)** only to carry the fix proposed in
[pkp/crossrefReferenceLinking#53](https://github.com/pkp/crossrefReferenceLinking/pull/53) while it
waits for review.

## License

GNU General Public License v3 — see [LICENSE](LICENSE).

---

## 🇧🇷 Português

**Este não é um plugin da OJSBR e não é para durar.** É o próprio plugin
`crossrefReferenceLinking` da PKP, versão 3.0.0.2, com **uma única alteração**: a correção do
[pkp/crossrefReferenceLinking#52](https://github.com/pkp/crossrefReferenceLinking/issues/52) — a
tarefa agendada que busca os DOIs das referências nunca é registrada quando o agendador roda pela
linha de comando, de modo que, em instalações conduzidas por cron, os DOIs das referências nunca
são recuperados.

A correção já está proposta no
[PR #53](https://github.com/pkp/crossrefReferenceLinking/pull/53). **Assim que a PKP aprovar e
publicar o plugin, esta versão é descontinuada** — este repositório será arquivado e cada
instalação deve voltar ao plugin da Galeria de Plugins da PKP. Nada aqui é funcionalidade nossa, e
pedido de funcionalidade não cabe aqui: cabe no repositório da PKP.

**Como saber se você é afetado:** com o plugin habilitado em alguma revista, rode
`php lib/pkp/tools/scheduler.php list | grep -i crossrefReferenceLinking` no diretório do OJS. Se
não sair nada, a tarefa nunca roda pelo cron naquela instalação.

**Instalação:** Administrador do site → **Configurações → Website → Plugins**, localize *Crossref
Reference Linking* entre os plugins instalados, escolha **Atualizar** e envie o `.tar.gz` das
[Releases](../../releases). A identidade do plugin é a mesma do original, então credenciais do
Crossref, configurações da revista e tudo que já está gravado em `crossref::*` nas submissões e
citações são preservados.

**Para voltar ao padrão:** quando a PKP publicar o plugin com a correção, a Galeria volta a
oferecer a atualização — basta aceitar.
