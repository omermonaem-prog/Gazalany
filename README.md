name: Build APK
on:
  workflow_dispatch:
  push:
    branches: [main, master]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: "17"
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: "3.27.4"
          channel: stable
      - name: Unzip project
        run: |
          Z="project.zip"
          [ -f "$Z" ] || Z="$(ls *.zip | head -1)"
          unzip -q "$Z" -d raw || true
          P="$(find raw -name pubspec.yaml | head -1)"
          mv "$(dirname "$P")" app
          rm -rf app/android
      - name: Fix Drift config
        run: |
          cat > app/build.yaml <<'EOF'
          targets:
            $default:
              builders:
                drift_dev:
                  options:
                    sql:
                      dialect: sqlite
          EOF
      - name: Fix int64 columns
        run: |
          sed -i 's/Int64Column/IntColumn/g; s/int64()/integer()/g' app/lib/core/database/app_database.dart
      - name: Fix imports
        run: |
          cat > fix.py <<'PY'
          import os, re, sys
          root = sys.argv[1]
          lib = os.path.join(root, 'lib')
          files = []
          for dp, _, fs in os.walk(lib):
              for f in fs:
                  if f.endswith('.dart'):
                      files.append(os.path.relpath(os.path.join(dp, f), lib).replace(os.sep, '/'))
          pat = re.compile(r"^(import|export)\s+'(\.[^']*)'", re.M)
          fixed = 0
          for rel in files:
              p = os.path.join(lib, rel)
              src = open(p, encoding='utf-8').read()
              d = os.path.dirname(p)
              def repl(m):
                  global fixed
                  t = m.group(2)
                  if os.path.exists(os.path.normpath(os.path.join(d, t))):
                      return m.group(0)
                  s = re.sub(r'^(\.\./|\./)+', '', t)
                  c = [f for f in files if f == s or f.endswith('/' + s)]
                  if len(c) > 1:
                      parts = rel.split('/')
                      feat = '/'.join(parts[:2]) + '/' if parts[0] == 'features' else ''
                      c = [x for x in c if feat and x.startswith(feat)] or c
                  if not c:
                      return m.group(0)
                  fixed += 1
                  return "%s 'package:my_finance/%s'" % (m.group(1), c[0])
              new = pat.sub(repl, src)
              if re.search(r'(?<![A-Za-z0-9_])Value\s*[(<]', new) and 'package:drift/drift.dart' not in new:
                  new = re.sub(r"^import ", "import 'package:drift/drift.dart' show Value;\nimport ", new, count=1, flags=re.M)
              if new != src:
                  open(p, 'w', encoding='utf-8').write(new)
          print('imports fixed:', fixed)
          PY
          python3 fix.py app
      - name: Create Android platform files
        working-directory: app
        run: flutter create . --org com.myfinance --project-name my_finance --platforms android
      - name: Set app name
        working-directory: app
        run: sed -i 's/android:label="[^"]*"/android:label="Jazlani"/' android/app/src/main/AndroidManifest.xml
      - name: Get packages
        working-directory: app
        run: flutter pub get
      - name: Generate Drift code
        working-directory: app
        run: dart run build_runner build --delete-conflicting-outputs
      - name: Analyze code
        working-directory: app
        run: |
          flutter analyze lib --no-fatal-infos --no-fatal-warnings > analyze.log 2>&1 || true
          grep -E "^ +error" analyze.log | sed 's/^ *//' | cut -c1-300 > errors.txt || true
          N=$(wc -l < errors.txt)
          echo "Total errors: $N"
          if [ "$N" -gt 0 ]; then
            exit 1
          fi
      - name: Upload error report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: analyze-report
          path: |
            app/errors.txt
            app/analyze.log
          if-no-files-found: ignore
      - name: Build APK
        working-directory: app
        run: flutter build apk --release
      - uses: actions/upload-artifact@v4
        with:
          name: jazlani-apk
          path: app/build/app/outputs/flutter-apk/app-release.apk
