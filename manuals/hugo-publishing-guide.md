# Hugo 블로그 포스트 작성 및 게시 매뉴얼

Hugo(PaperMod 테마 기준)에서 새로운 글을 작성하고 게시하는 전체 과정입니다.

---

## 1. 새 포스트 생성 (Create)
터미널에서 아래 명령어를 실행하여 새 마크다운 파일을 생성합니다.
```bash
hugo new posts/파일명.md
```
- **생성 위치**: `content/posts/파일명.md`
- **참고**: `archetypes/default.md` 설정에 따라 기본적인 Front Matter(메타데이터)가 자동 생성됩니다.

## 2. 내용 작성 및 설정 (Edit)
생성된 파일을 에디터로 열어 내용을 수정합니다.
- **title**: 글의 제목을 입력합니다.
- **date**: 파일 생성 시각이 자동으로 기록됩니다.
- **draft**: 기본적으로 `true`로 설정됩니다. `true` 상태에서는 실제 사이트에 게시되지 않으므로, 작성이 완료되면 `false`로 변경해야 합니다.

## 3. 로컬 미리보기 (Preview)
수정한 내용이 의도대로 나오는지 로컬 서버를 통해 확인합니다.
```bash
hugo server -D
```
- **-D 옵션**: `draft: true` 상태인 글도 포함하여 보여줍니다.
- **접속 주소**: [http://localhost:1313](http://localhost:1313)

## 4. 빌드 및 배포 (Deploy)
작성이 완료되고 검토가 끝났다면 실제 서버에 반영합니다.

1. **상태 확인**: `draft: false`로 변경했는지 확인합니다.
2. **변경 사항 저장 및 푸시**:
   ```bash
   git add .
   git commit -m "Add new post: 파일명"
   git push origin main
   ```
- **자동 배포**: 현재 프로젝트는 `.github/workflows/deploy.yml` 설정에 의해 GitHub에 푸시하면 자동으로 빌드 및 배포가 진행됩니다.
