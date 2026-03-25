# Cobra (Go)

Framework-specific patterns for applying the agent-friendly CLI checklist in Go. Cobra powers kubectl, Docker CLI, Hugo, and GitHub CLI (173K+ projects).

## Setup

```bash
go get -u github.com/spf13/cobra@latest
go install github.com/spf13/cobra-cli@latest
```

## Non-interactive flags (Checklist 1)

```go
package cmd

import (
    "fmt"
    "github.com/spf13/cobra"
)

var deployCmd = &cobra.Command{
    Use:   "deploy",
    Short: "Deploy an application to a target environment",
    RunE: func(cmd *cobra.Command, args []string) error {
        env, _ := cmd.Flags().GetString("env")
        tag, _ := cmd.Flags().GetString("tag")
        force, _ := cmd.Flags().GetBool("force")

        if !force {
            fmt.Printf("Deploy %s to %s? [y/N] ", tag, env)
            // read confirmation
        }
        return runDeploy(env, tag)
    },
}

func init() {
    deployCmd.Flags().StringP("env", "e", "", "target environment (staging, production)")
    deployCmd.Flags().StringP("tag", "t", "latest", "image tag")
    deployCmd.Flags().Bool("force", false, "skip confirmation")
    deployCmd.MarkFlagRequired("env")
}
```

## Structured output (Checklist 2)

Add a persistent `--output` flag at the root level:

```go
var outputFormat string

var rootCmd = &cobra.Command{
    Use: "mycli",
}

func init() {
    rootCmd.PersistentFlags().StringVarP(&outputFormat, "output", "o", "text", "output format (text, json)")
}

func printResult(data interface{}, humanMsg string) {
    if outputFormat == "json" {
        result := map[string]interface{}{
            "status": "success",
            "data":   data,
            "error":  nil,
        }
        enc := json.NewEncoder(os.Stdout)
        enc.SetIndent("", "  ")
        enc.Encode(result)
    } else {
        fmt.Println(humanMsg)
    }
}
```

## Help with examples (Checklist 3)

Cobra has a dedicated `Example` field:

```go
var deployCmd = &cobra.Command{
    Use:   "deploy",
    Short: "Deploy an application to a target environment",
    Example: `  mycli deploy --env staging
  mycli deploy --env production --tag v1.2.3
  mycli deploy --env staging --force`,
    RunE: func(cmd *cobra.Command, args []string) error {
        return nil
    },
}
```

## Actionable errors (Checklist 4)

Return errors from `RunE` with context:

```go
var deployCmd = &cobra.Command{
    Use: "deploy",
    RunE: func(cmd *cobra.Command, args []string) error {
        tag, _ := cmd.Flags().GetString("tag")
        if tag == "" {
            return fmt.Errorf("no image tag specified\n\n  mycli deploy --env staging --tag <image-tag>\n  Available tags: mycli build list --output tags")
        }
        return runDeploy(tag)
    },
}
```

## Exit codes (Checklist 8)

```go
const (
    ExitSuccess     = 0
    ExitGeneral     = 1
    ExitUsage       = 2
    ExitConfig      = 3
    ExitNotFound    = 4
    ExitPermission  = 5
    ExitNetwork     = 10
)

func main() {
    if err := rootCmd.Execute(); err != nil {
        switch {
        case errors.Is(err, ErrConfig):
            os.Exit(ExitConfig)
        case errors.Is(err, ErrNotFound):
            os.Exit(ExitNotFound)
        default:
            os.Exit(ExitGeneral)
        }
    }
}
```

## Stdin support (Checklist 9)

```go
var configImportCmd = &cobra.Command{
    Use: "import",
    RunE: func(cmd *cobra.Command, args []string) error {
        useStdin, _ := cmd.Flags().GetBool("stdin")

        var reader io.Reader
        if useStdin {
            reader = os.Stdin
        } else {
            filePath, _ := cmd.Flags().GetString("file")
            f, err := os.Open(filePath)
            if err != nil {
                return fmt.Errorf("cannot open %s: %w", filePath, err)
            }
            defer f.Close()
            reader = f
        }

        data, err := io.ReadAll(reader)
        if err != nil {
            return err
        }
        return importConfig(data)
    },
}
```

## Env var support (Checklist 10)

Cobra integrates with Viper for env var binding:

```go
import "github.com/spf13/viper"

func init() {
    deployCmd.Flags().StringP("token", "", "", "API token [$MYCLI_TOKEN]")
    viper.BindPFlag("token", deployCmd.Flags().Lookup("token"))
    viper.BindEnv("token", "MYCLI_TOKEN")
}
```

Without Viper, check env vars manually:

```go
func getToken(cmd *cobra.Command) string {
    token, _ := cmd.Flags().GetString("token")
    if token != "" {
        return token
    }
    return os.Getenv("MYCLI_TOKEN")
}
```

## Subcommands (Checklist 11)

```go
var serviceCmd = &cobra.Command{Use: "service", Short: "Manage services"}
var serviceCreateCmd = &cobra.Command{Use: "create", Short: "Create a service", RunE: createService}
var serviceListCmd = &cobra.Command{Use: "list", Short: "List services", RunE: listServices}
var serviceDeleteCmd = &cobra.Command{Use: "delete", Short: "Delete a service", RunE: deleteService}

func init() {
    rootCmd.AddCommand(serviceCmd)
    serviceCmd.AddCommand(serviceCreateCmd, serviceListCmd, serviceDeleteCmd)
}
```
