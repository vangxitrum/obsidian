# aioz-template — memory index

Project: `/home/tuan/work/templates/aioz-template` — the gonew-compatible Go service template (module `gitlab.internal/tuan.quang.tran/aioz-template`) composing aioz-config, aioz-logger and aioz-stats.

- [[service-template-scaffold]] — the shape of the template, the gonew contract (no non-Go file may hardcode the module path), and the exact config/logger bootstrap ordering in `run`.
- [[cobra-completion-wiring]] — cobra's free `completion` cmd still file-completes every string flag; how the template fixes that, and the argv-named-command and `--defaults` panic it exposed.
