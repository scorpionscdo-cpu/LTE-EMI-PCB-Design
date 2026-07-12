# Supabase MCP 설정

이 저장소에는 Supabase MCP 서버가 `.mcp.json`에 설정되어 있습니다.
Claude Code가 이 저장소를 열면 자동으로 서버를 인식합니다.

## 사전 준비

`.mcp.json`은 다음 두 환경변수를 사용합니다. 사용하기 전에 셸 환경에 설정하세요.

```bash
# https://supabase.com/dashboard/account/tokens 에서 발급
export SUPABASE_ACCESS_TOKEN="sbp_..."

# 대상 프로젝트의 project ref (Dashboard > Project Settings > General)
export SUPABASE_PROJECT_REF="your-project-ref"
```

> 보안상 액세스 토큰은 `.mcp.json`에 직접 넣지 않고 환경변수로 주입합니다.
> 토큰을 저장소에 커밋하지 마세요.

## 옵션 설명

- `--read-only`: 읽기 전용 모드. 데이터베이스에 대한 쓰기/DDL 작업을 차단하여
  안전하게 조회만 수행합니다. 쓰기가 필요하면 이 플래그를 제거하세요.
- `--project-ref`: 단일 프로젝트로 범위를 제한합니다. 여러 프로젝트에
  접근하려면 이 인자를 제거하세요.

## 사용 확인

Claude Code에서 `/mcp` 명령으로 `supabase` 서버가 연결되었는지 확인할 수 있습니다.
