# Configurações do meui VSCode

Aqui estão algumas das minhas configurações favoritas do VSCode para melhorar a produtividade e a experiência de codificação:

# settings.json

```json
{
     "editor.lineHeight": 24,
     "editor.tabSize": 5,
     "editor.insertSpaces": true,
     "editor.renderWhitespace": "all",
     "editor.rulers": [
          80,
          100,
          120
     ],
     "tabnine.experimentalAutoImports": true,
     "files.associations": {
          ".sequelizerc": "javascript",
          ".stylelintrc": "json",
          "*.tsx": "typescriptreact",
          ".env.*": "dotenv",
          ".prettierrc": "json",
          "*.yaml.tpl": "yaml",
          "*.yml.tpl": "yaml",
          "*yaml.tftpl": "yaml",
          "*yml.tftpl": "yaml",
          "*.tftpl": "yaml",
          "**/*.tpl": "yaml",
     },
     // Python
     "python.linting.pylintEnabled": false,
     "python.linting.flake8Enabled": true,
     "python.linting.enabled": true,
     "python.formatting.provider": "black",

     // yaml and yml
     "[yaml]": {
          "editor.tabSize": 2,
          "editor.insertSpaces": true,
          "editor.formatOnSave": true
     },

     "emmet.includeLanguages": {
          "javascript": "javascriptreact"
     },
     "indentRainbow.includedLanguages": ["html"],
     "indentRainbow.lightIndicatorStyleLineWidth": 4,
     "liveServer.settings.donotVerifyTags": true,
     "workbench.startupEditor": "none",
     "vsicons.dontShowNewVersionMessage": true,
     "explorer.confirmDelete": false,
     "vs-kubernetes": {
          "vscode-kubernetes.minikube-path.linux": "/home/${USER}/.local/state/vs-kubernetes/tools/minikube/linux-amd64/minikube"
     },
     "workbench.iconTheme": "material-icon-theme",
     "workbench.colorTheme": "Electron",
     "github.copilot.nextEditSuggestions.enabled": true,
     "[plaintext]": {
     "editor.unicodeHighlight.ambiguousCharacters": false,
     "editor.unicodeHighlight.invisibleCharacters": false
     },
     "github.copilot.enable": {
     "*": true,
     "plaintext": true,
     "markdown": true,
     "scminput": false
     },
     "python.analysis.typeCheckingMode": "basic",
     "[python]": {
     "diffEditor.ignoreTrimWhitespace": false,
     "editor.defaultColorDecorators": "never",
     "editor.formatOnType": true,
     "editor.wordBasedSuggestions": "off"
     }
}
```
