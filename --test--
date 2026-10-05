import { promises as fs } from 'node:fs'
import path from 'node:path'
import { fileURLToPath } from 'node:url'
import { describe, expect, it } from 'vitest'

// Dockerfile.dev does `COPY . /app-template`, so anything the build context
// keeps is baked into the preview image. A nested node_modules that survives
// shadows the hoisted /node_modules during resolution, and `**/dist` strips
// each package's build output on the way in — so a partially-copied nested
// install is worse than no install: `vite.config.ts` imports source-mapper,
// resolves @babel/generator to the stripped copy, and the dev server dies on
// `Cannot find module '.../@jridgewell/gen-mapping/dist/gen-mapping.umd.js'`.
// These assertions read the real .dockerignore rather than a fixture, because
// the pairing of the two patterns is what makes a nested install fatal.
const TEMPLATE_DIR: string = path.resolve(path.dirname(fileURLToPath(import.meta.url)), '..')
const DOCKERIGNORE_FILE: string = path.join(TEMPLATE_DIR, '.dockerignore')

// Packages whose own node_modules a developer populates locally; each is
// gitignored, so CI never sees one and only local builds bake them in.
const NESTED_PACKAGES: readonly string[] = ['source-mapper', 'dev-tools', 'content-plugin']

const REGEXP_METACHARACTERS: RegExp = /[.+^${}()|[\]\\]/g

function readPatterns(contents: string): string[] {
  return contents
    .split('\n')
    .map((line: string): string => line.trim())
    .filter((line: string): boolean => line.length > 0 && !line.startsWith('#'))
}

// Mirrors Docker's build-context matching for the pattern forms this file
// uses: `**` spans any number of segments, `*` and `?` stay within one.
function patternToRegExp(pattern: string): RegExp {
  const segments: string[] = pattern.split('/')
  const source: string = segments
    .map((segment: string, index: number): string => {
      const isLast: boolean = index === segments.length - 1
      if (segment === '**') return isLast ? '.*' : '(?:[^/]+/)*'
      const escaped: string = segment
        .replace(REGEXP_METACHARACTERS, '\\$&')
        .replace(/\*/g, '[^/]*')
        .replace(/\?/g, '[^/]')
      return isLast ? escaped : `${escaped}/`
    })
    .join('')
  return new RegExp(`^${source}$`) // nosemgrep: javascript.lang.security.audit.detect-non-literal-regexp.detect-non-literal-regexp
}

// Docker drops a directory's whole subtree, so a path is out of the context
// when it matches a pattern or descends from something that does.
function isExcluded(contextPath: string, patterns: readonly RegExp[]): boolean {
  const segments: string[] = contextPath.split('/')
  return segments.some((_segment: string, index: number): boolean => {
    const ancestor: string = segments.slice(0, index + 1).join('/')
    return patterns.some((pattern: RegExp): boolean => pattern.test(ancestor))
  })
}

async function loadPatterns(): Promise<{ raw: string[]; compiled: RegExp[] }> {
  const contents: string = await fs.readFile(DOCKERIGNORE_FILE, 'utf8')
  const raw: string[] = readPatterns(contents)
  return { raw, compiled: raw.map(patternToRegExp) }
}

describe('v8 template .dockerignore', () => {
  it('declares no negation patterns, which the matcher here does not model', async () => {
    const { raw } = await loadPatterns()
    expect(raw.filter((pattern: string): boolean => pattern.startsWith('!'))).toEqual([])
  })

  it('keeps every nested node_modules out of the build context', async () => {
    const { compiled } = await loadPatterns()
    for (const pkg of NESTED_PACKAGES) {
      expect(isExcluded(`${pkg}/node_modules`, compiled), pkg).toBe(true)
    }
  })

  it('excludes a nested package outright rather than shipping it without its dist', async () => {
    const { compiled } = await loadPatterns()
    const stripped: string = 'source-mapper/node_modules/@jridgewell/gen-mapping/dist/gen-mapping.umd.js'
    const manifest: string = 'source-mapper/node_modules/@jridgewell/gen-mapping/package.json'
    expect(isExcluded(stripped, compiled)).toBe(true)
    expect(isExcluded(manifest, compiled)).toBe(true)
  })

  it('still excludes the top-level node_modules', async () => {
    const { compiled } = await loadPatterns()
    expect(isExcluded('node_modules/vite/package.json', compiled)).toBe(true)
  })

  it("still excludes a locally built package's dist", async () => {
    const { compiled } = await loadPatterns()
    expect(isExcluded('content-plugin/dist/index.js', compiled)).toBe(true)
    expect(isExcluded('dist/assets/index.js', compiled)).toBe(true)
  })

  it('still excludes tests and specs', async () => {
    const { compiled } = await loadPatterns()
    expect(isExcluded('src/lib/__tests__/thing.test.ts', compiled)).toBe(true)
    expect(isExcluded('content-plugin/parse.spec.ts', compiled)).toBe(true)
  })

  it('keeps the template sources the image needs', async () => {
    const { compiled } = await loadPatterns()
    for (const kept of ['vite.config.ts', 'package.json', 'src/main.tsx', 'source-mapper/src/index.ts']) {
      expect(isExcluded(kept, compiled), kept).toBe(false)
    }
  })
})
