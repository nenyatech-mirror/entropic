# OpenClaw Core Runtime Container
# Hardened, non-root, read-only filesystem

FROM node:22-bookworm-slim

# Install runtime deps and create directories
RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates \
    curl \
    git \
    pandoc \
    python3 \
    python3-pip \
    python3-venv \
    sqlite3 \
    unzip \
    # Headless Chromium runtime dependencies (patchright)
    libasound2 \
    libatk-bridge2.0-0 \
    libatk1.0-0 \
    libcups2 \
    libdbus-1-3 \
    libdrm2 \
    libgbm1 \
    libglib2.0-0 \
    libgtk-3-0 \
    libnspr4 \
    libnss3 \
    libpango-1.0-0 \
    libx11-6 \
    libxcomposite1 \
    libxdamage1 \
    libxext6 \
    libxfixes3 \
    libxkbcommon0 \
    libxrandr2 \
    fonts-liberation \
    fonts-noto-color-emoji \
    # Additional fonts for realistic browser fingerprint
    fonts-noto-core \
    fonts-dejavu-core \
    # Video codec libraries (h264 support for Chromium)
    libavcodec59 \
    libavformat59 \
    libavutil57 \
    xvfb && \
    rm -rf /var/lib/apt/lists/* && \
    # Make `python` resolve to python3 so skills can call either name
    ln -sfn /usr/bin/python3 /usr/local/bin/python && \
    mkdir -p /data /data/python-envs /home/node/.openclaw /home/node/.bun /home/node/.cache/qmd && \
    ln -sfn /data/qmd-models /home/node/.cache/qmd/models && \
    chown -R node:node /data /home/node/.openclaw /home/node/.bun /home/node/.cache

ARG ENTROPIC_INSTALL_DOCLING=0
RUN python3 -m pip install --no-cache-dir --break-system-packages \
    openpyxl==3.1.5 \
    pytest==8.3.5 \
    python-pptx==1.0.2 && \
    if [ "$ENTROPIC_INSTALL_DOCLING" = "1" ]; then \
      python3 -m pip install --no-cache-dir --break-system-packages 'docling>=2.64,<3'; \
    fi

# Install patchright's patched Chromium:
# 1. Symlink to /usr/bin/chromium (OpenClaw's findChromeExecutableLinux() searches there)
# 2. Copy browsers to /opt/patchright-browsers/ so entrypoint can symlink into
#    PLAYWRIGHT_BROWSERS_PATH (/data/playwright/) for patchright's chromium.launch()
RUN npx patchright@latest install --no-shell chromium && \
    cp -a /root/.cache/ms-playwright /opt/patchright-browsers && \
    chmod -R a+rX /opt/patchright-browsers && \
    CHROME_BIN=$(find /opt/patchright-browsers -name 'chrome' -type f | head -1) && \
    if [ -n "$CHROME_BIN" ]; then ln -sfn "$CHROME_BIN" /usr/bin/chromium; fi && \
    rm -rf /root/.cache/ms-playwright /tmp/.npm /tmp/patchright-*

ENV BUN_INSTALL=/data/.bun
# Python user installs land in /data/.local so they survive container restarts
# (the node user's HOME is /home/node but /data is the persistent volume)
ENV PYTHONUSERBASE=/data/.local
ENV PATH="/data/.bun/bin:/data/.local/bin:${PATH}"

WORKDIR /app

# Copy OpenClaw package metadata and deps first (stable for caching)
COPY --chown=node:node package.json ./package.json
COPY --chown=node:node node_modules ./node_modules
COPY --chown=node:node all-extension-runtime-deps.txt ./all-extension-runtime-deps.txt
COPY --chown=node:node docs ./docs
COPY --chown=node:node extensions ./extensions
COPY --chown=node:node browser-service ./browser-service

# Copy OpenClaw dist last (changes most often)
COPY --chown=node:node dist ./dist

# Copy bundled skills (seeded into /data/skills/ at startup by entrypoint)
COPY --chown=node:node bundled-skills /app/bundled-skills

# Upstream's compiled root dist can statically import SDKs owned by optional
# channel extensions, even when Nova chooses not to ship those extensions in the
# runtime image. Install a compatibility dependency list generated from all
# upstream extension manifests so /app/dist stays resolvable without modifying
# upstream OpenClaw itself.
RUN EXT_DEPS="$(tr '\n' ' ' </app/all-extension-runtime-deps.txt)" && \
    if [ -n "$EXT_DEPS" ]; then \
      echo "Installing upstream extension compatibility deps into root node_modules..." && \
      npm install --omit=dev --ignore-scripts --legacy-peer-deps --package-lock=false --no-save $EXT_DEPS; \
    fi

# Install bundled skill dependencies (runs as root before USER node)
RUN for skill in /app/bundled-skills/*/; do \
      if [ -f "$skill/package.json" ]; then \
        echo "Installing deps for $(basename "$skill")..." && \
        cd "$skill" && npm install --omit=dev && cd /; \
      fi; \
    done

RUN if [ -f /app/browser-service/package.json ]; then \
      cd /app/browser-service && npm install --omit=dev && cd /; \
    fi

RUN mkdir -p /app/node_modules/osc-progress && \
    printf '{"name":"osc-progress","type":"module","exports":"./index.js"}\n' > /app/node_modules/osc-progress/package.json && \
    printf 'export function supportsOscProgress() { return false; }\nexport function createOscProgressController() { return { start() {}, update() {}, done() {}, fail() {} }; }\n' > /app/node_modules/osc-progress/index.js

# The compiled root dist imports Discord channel helpers directly. The Discord
# extension vendors @buape/carbon under its own node_modules, so expose that
# package at the root resolver location too.
RUN if [ -d /app/dist/extensions/discord/node_modules/@buape/carbon ]; then \
      mkdir -p /app/node_modules/@buape && \
      ln -sfn /app/dist/extensions/discord/node_modules/@buape/carbon /app/node_modules/@buape/carbon; \
      node -e 'const fs=require("fs"); const p="/app/dist/extensions/discord/node_modules/@buape/carbon/package.json"; const pkg=JSON.parse(fs.readFileSync(p,"utf8")); for (const key of [".","./*","./adapters/*"]) { if (pkg.exports?.[key]?.import && !pkg.exports[key].require) pkg.exports[key].require = pkg.exports[key].import; } fs.writeFileSync(p, JSON.stringify(pkg,null,2)+"\n");'; \
    fi

RUN PI_CODING_INDEX="/app/node_modules/@mariozechner/pi-coding-agent/dist/index.js" && \
    if ! grep -q 'Legacy Entropic compatibility exports' "$PI_CODING_INDEX"; then \
      printf '\n// Legacy Entropic compatibility exports.\nimport { createCodingTools as __entropicCreateCodingTools, createReadTool as __entropicCreateReadTool } from "./core/sdk.js";\nexport const codingTools = __entropicCreateCodingTools(process.cwd());\nexport const readTool = __entropicCreateReadTool(process.cwd());\n' >> "$PI_CODING_INDEX"; \
    fi

# Some OpenAI-compatible provider paths can return a visible XML/JSON tool-call
# transcript even when tools were advertised. Recover that into an actual pi-ai
# toolCall block so OpenClaw executes the tool instead of showing raw syntax.
RUN node <<'NODE'
const fs = require("fs");
const path = require("path");

const distDir = "/app/dist";
const runner = fs.readdirSync(distDir).find((file) => /^pi-embedded-runner-.*\.js$/.test(file));
if (!runner) throw new Error("pi-embedded-runner bundle not found");
const filePath = path.join(distDir, runner);
let source = fs.readFileSync(filePath, "utf8");
const existingHelperStart = source.indexOf("// Entropic raw text tool-call recovery.");
if (existingHelperStart >= 0) {
  const existingHelperEnd = source.indexOf("function wrapStreamFnTrimToolCallNames(baseFn, allowedToolNames) {", existingHelperStart);
  if (existingHelperEnd < 0) throw new Error("Existing Entropic helper end marker not found");
  source = source.slice(0, existingHelperStart) + source.slice(existingHelperEnd);
}
const helper = String.raw`
// Entropic raw text tool-call recovery.
function parseEntropicRawTextToolCalls(text, allowedToolNames) {
	const calls = [];
	if (typeof text !== "string" || !/<tool_call/i.test(text)) return calls;
	const blockRe = /<tool_call>\s*({[\s\S]*?})\s*<\/tool_call>/gi;
	let match;
	let index = 0;
	while ((match = blockRe.exec(text))) {
		try {
			const parsed = JSON.parse(match[1]);
			const rawName = typeof parsed?.name === "string" ? parsed.name : typeof parsed?.function?.name === "string" ? parsed.function.name : "";
			const name = resolveExactAllowedToolName(rawName, allowedToolNames);
			if (!name) continue;
			const args = parsed?.arguments ?? parsed?.function?.arguments ?? {};
			const normalizedArgs = typeof args === "string" ? JSON.parse(args || "{}") : args && typeof args === "object" ? args : {};
			calls.push({
				type: "toolCall",
				id: "call_entropic_raw_" + ++index,
				name,
				arguments: normalizedArgs
			});
		} catch {}
	}
	return calls;
}
function recoverEntropicRawTextToolCallsInMessage(message, allowedToolNames) {
	if (!message || typeof message !== "object" || !Array.isArray(message.content)) return false;
	const textBlocks = message.content.filter((block) => block && typeof block === "object" && block.type === "text" && typeof block.text === "string");
	const rawText = textBlocks.map((block) => block.text).join("\n");
	const calls = parseEntropicRawTextToolCalls(rawText, allowedToolNames);
	if (calls.length === 0) return false;
	const cleanedText = rawText.replace(/<tool_call>\s*{[\s\S]*?}\s*<\/tool_call>/gi, "").trim();
	const nextContent = [];
	if (cleanedText && !/couldn['’]t execute|could not execute|can['’]t reliably|without risking a stale answer/i.test(cleanedText)) {
		nextContent.push({ type: "text", text: cleanedText });
	}
	nextContent.push(...calls);
	message.content = nextContent;
	message.stopReason = "toolUse";
	return true;
}
function wrapStreamFnRecoverEntropicRawTextToolCalls(baseFn, allowedToolNames) {
	return (model, context, options) => {
		const maybeStream = baseFn(model, context, options);
		const wrap = (stream) => {
			if (!stream || typeof stream !== "object") return stream;
			const originalResult = typeof stream.result === "function" ? stream.result.bind(stream) : null;
			if (originalResult) {
				stream.result = async () => {
					const message = await originalResult();
					recoverEntropicRawTextToolCallsInMessage(message, allowedToolNames);
					return message;
				};
			}
			const originalIterator = typeof stream[Symbol.asyncIterator] === "function" ? stream[Symbol.asyncIterator].bind(stream) : null;
			if (originalIterator) {
				stream[Symbol.asyncIterator] = function() {
					const iterator = originalIterator();
					return {
						async next() {
							const result = await iterator.next();
							if (!result.done && result.value && typeof result.value === "object") {
								recoverEntropicRawTextToolCallsInMessage(result.value.partial, allowedToolNames);
								recoverEntropicRawTextToolCallsInMessage(result.value.message, allowedToolNames);
							}
							return result;
						},
						async return(value) {
							return iterator.return?.(value) ?? { done: true, value };
						},
						async throw(error) {
							return iterator.throw?.(error) ?? Promise.reject(error);
						}
					};
				};
			}
			return stream;
		};
		if (maybeStream && typeof maybeStream === "object" && "then" in maybeStream) return Promise.resolve(maybeStream).then(wrap);
		return wrap(maybeStream);
	};
}
function entropicContextLatestUserText(context) {
	const messages = Array.isArray(context?.messages) ? context.messages : [];
	for (let i = messages.length - 1; i >= 0; i--) {
		const message = messages[i];
		if (!message || typeof message !== "object" || message.role !== "user") continue;
		const content = message.content;
		if (typeof content === "string") return content;
		if (Array.isArray(content)) {
			return content.map((block) => block && typeof block === "object" && block.type === "text" ? block.text || "" : "").join("\n");
		}
	}
	return "";
}
function entropicLatestUserIndex(context) {
	const messages = Array.isArray(context?.messages) ? context.messages : [];
	for (let i = messages.length - 1; i >= 0; i--) {
		const message = messages[i];
		if (message && typeof message === "object" && message.role === "user") return i;
	}
	return -1;
}
function entropicMessageHasBrowserToolUse(message) {
	const content = Array.isArray(message?.content) ? message.content : [];
	return content.some((block) => block && typeof block === "object" && block.type === "toolCall" && block.name === "browser");
}
function entropicBrowserToolAlreadyUsedAfterLatestUser(context) {
	const messages = Array.isArray(context?.messages) ? context.messages : [];
	const latestUserIndex = entropicLatestUserIndex(context);
	if (latestUserIndex < 0) return false;
	return messages.slice(latestUserIndex + 1).some(entropicMessageHasBrowserToolUse);
}
function hasEntropicAllowedTool(allowedToolNames, name) {
	if (allowedToolNames?.has?.(name)) return true;
	if (Array.isArray(allowedToolNames)) return allowedToolNames.includes(name);
	return false;
}
function shouldEntropicForceBrowserTool(context, allowedToolNames) {
	if (!hasEntropicAllowedTool(allowedToolNames, "browser")) return false;
	if (entropicBrowserToolAlreadyUsedAfterLatestUser(context)) return false;
	const text = entropicContextLatestUserText(context).toLowerCase();
	if (!text) return false;
	if (/\b(weather|temperature|forecast|rain|storm|humidity|wind|air quality|aqi)\b/.test(text)) return true;
	if (/\b(stock|stocks|market|markets|s&p|sp500|nasdaq|dow|price|prices|quote|closed|close today)\b/.test(text)) return true;
	if (/\b(news|latest|current|right now|today|tonight|tomorrow|live|lookup|look up|web|internet)\b/.test(text)) return true;
	return false;
}
function entropicBrowserUrlForCurrentInfo(context) {
	const rawText = entropicContextLatestUserText(context).replace(/Sender \(untrusted metadata\):[\s\S]*?\]\s*/m, "").trim();
	const query = rawText || "current information";
	return "https://www.google.com/search?q=" + encodeURIComponent(query);
}
function createEntropicSyntheticBrowserToolResponse(model, context) {
	const message = {
		role: "assistant",
		content: [{
			type: "toolCall",
			id: "call_entropic_browser_current_info",
			name: "browser",
			arguments: {
				action: "open",
				url: entropicBrowserUrlForCurrentInfo(context)
			}
		}],
		stopReason: "toolUse",
		api: model?.api ?? "",
		provider: model?.provider ?? "",
		model: model?.id ?? "",
		usage: {
			input: 0,
			output: 0,
			cacheRead: 0,
			cacheWrite: 0,
			totalTokens: 0,
			cost: {
				input: 0,
				output: 0,
				cacheRead: 0,
				cacheWrite: 0,
				total: 0
			}
		},
		timestamp: Date.now()
	};
	return {
		async *[Symbol.asyncIterator]() {},
		result: async () => message
	};
}
function wrapStreamFnForceBrowserForCurrentInfo(baseFn, allowedToolNames) {
	return (model, context, options) => {
		if (shouldEntropicForceBrowserTool(context, allowedToolNames)) {
			return createEntropicSyntheticBrowserToolResponse(model, context);
		}
		return baseFn(model, context, options);
	};
}
`;
source = source.replace("function wrapStreamFnTrimToolCallNames(baseFn, allowedToolNames) {", `${helper}\nfunction wrapStreamFnTrimToolCallNames(baseFn, allowedToolNames) {`);
if (!source.includes("wrapStreamFnForceBrowserForCurrentInfo(activeSession.agent.streamFn, allowedToolNames)")) {
  source = source.replace(
    "activeSession.agent.streamFn = wrapStreamFnSanitizeMalformedToolCalls(activeSession.agent.streamFn, allowedToolNames, transcriptPolicy);",
    "activeSession.agent.streamFn = wrapStreamFnSanitizeMalformedToolCalls(activeSession.agent.streamFn, allowedToolNames, transcriptPolicy);\n\t\t\tactiveSession.agent.streamFn = wrapStreamFnForceBrowserForCurrentInfo(activeSession.agent.streamFn, allowedToolNames);\n\t\t\tactiveSession.agent.streamFn = wrapStreamFnRecoverEntropicRawTextToolCalls(activeSession.agent.streamFn, allowedToolNames);"
  );
}
fs.writeFileSync(filePath, source);
NODE

# Venice/OpenRouter rejects OpenAI-compatible completion requests with an empty
# tools array. OpenClaw can create tool-history turns for client-side tools such
# as browser, so omit the field instead of sending `tools: []`.
RUN node <<'NODE'
const fs = require("fs");
const path = require("path");

const distDir = "/app/dist";
const streamFile = fs.readdirSync(distDir).find((file) => /^anthropic-vertex-stream-.*\.js$/.test(file));
if (!streamFile) throw new Error("anthropic-vertex-stream bundle not found");
const filePath = path.join(distDir, streamFile);
let source = fs.readFileSync(filePath, "utf8");
source = source.replace(
	"} else if (hasToolHistory(context.messages)) params.tools = [];",
	"}"
);
fs.writeFileSync(filePath, source);
NODE

RUN chmod +x /app/browser-service/office_tools.py && \
    ln -sfn /app/browser-service/office_tools.py /usr/local/bin/entropic-office

# Fix clipboard module for ARM64 Linux (create stub)
RUN echo "module.exports = { read: () => '', write: () => {}, readSync: () => '', writeSync: () => {}, clear: () => {}, clearSync: () => {} };" > /app/node_modules/.pnpm/@mariozechner+clipboard@0.3.0/node_modules/@mariozechner/clipboard/index.js || true

# Fix sharp native bindings for Linux when node_modules were materialized outside
# the final image. Pick the linux binding that matches the image architecture
# instead of assuming an arm64 host.
RUN SHARP_PNPM="$(find /app/node_modules/.pnpm -maxdepth 1 -name 'sharp@*' -type d | head -1)" && \
    SHARP_VER="$(basename "$SHARP_PNPM" | sed 's/sharp@//')" && \
    SHARP_IMG_DIR="$SHARP_PNPM/node_modules/@img" && \
    TARGET_ARCH="$(dpkg --print-architecture)" && \
    case "$TARGET_ARCH" in \
      amd64) SHARP_SUFFIX="linux-x64" ;; \
      arm64) SHARP_SUFFIX="linux-arm64" ;; \
      *) SHARP_SUFFIX="" ;; \
    esac && \
    LIBVIPS_VER="$(find /app/node_modules/.pnpm -maxdepth 1 -type d -name "@img+sharp-libvips-*@*" | head -1 | sed 's/.*@//')" && \
    echo "sharp=$SHARP_VER libvips=$LIBVIPS_VER arch=$TARGET_ARCH suffix=$SHARP_SUFFIX img_dir=$SHARP_IMG_DIR" && \
    if [ -d "$SHARP_IMG_DIR" ] && [ -n "$SHARP_VER" ] && [ -n "$SHARP_SUFFIX" ]; then \
      cd /tmp && mkdir sharp-fix && cd sharp-fix && \
      npm init -y --silent 2>/dev/null && \
      npm install "@img/sharp-$SHARP_SUFFIX@$SHARP_VER" "@img/sharp-libvips-$SHARP_SUFFIX@${LIBVIPS_VER:-1.2.4}" 2>&1 && \
      ls -la node_modules/@img/ && \
      cp -a "node_modules/@img/sharp-$SHARP_SUFFIX" "$SHARP_IMG_DIR/sharp-$SHARP_SUFFIX" && \
      cp -a "node_modules/@img/sharp-libvips-$SHARP_SUFFIX" "$SHARP_IMG_DIR/sharp-libvips-$SHARP_SUFFIX" && \
      echo "Installed sharp $SHARP_SUFFIX bindings into $SHARP_IMG_DIR" && \
      cd / && rm -rf /tmp/sharp-fix; \
    fi

# Fix @snazzah/davey native bindings for Linux when node_modules were
# materialized on macOS. The optional platform package is selected at install
# time, so copying host node_modules into the runtime image can miss the Linux
# binary entirely and cause the gateway to crash on boot.
RUN DAVEY_PNPM="$(find /app/node_modules/.pnpm -maxdepth 1 -name '@snazzah+davey@*' -type d | head -1)" && \
    DAVEY_VER="$(basename "$DAVEY_PNPM" | sed 's/@snazzah+davey@//')" && \
    DAVEY_SCOPE_DIR="/app/node_modules/.pnpm/node_modules/@snazzah" && \
    TARGET_ARCH="$(dpkg --print-architecture)" && \
    case "$TARGET_ARCH" in \
      amd64) DAVEY_SUFFIX="linux-x64-gnu" ;; \
      arm64) DAVEY_SUFFIX="linux-arm64-gnu" ;; \
      *) DAVEY_SUFFIX="" ;; \
    esac && \
    echo "davey=$DAVEY_VER arch=$TARGET_ARCH suffix=$DAVEY_SUFFIX scope_dir=$DAVEY_SCOPE_DIR" && \
    if [ -d "$DAVEY_SCOPE_DIR" ] && [ -n "$DAVEY_VER" ] && [ -n "$DAVEY_SUFFIX" ]; then \
      cd /tmp && mkdir davey-fix && cd davey-fix && \
      npm init -y --silent 2>/dev/null && \
      npm install "@snazzah/davey-$DAVEY_SUFFIX@$DAVEY_VER" 2>&1 && \
      mkdir -p "$DAVEY_SCOPE_DIR" && \
      cp -a "node_modules/@snazzah/davey-$DAVEY_SUFFIX" "$DAVEY_SCOPE_DIR/davey-$DAVEY_SUFFIX" && \
      echo "Installed davey $DAVEY_SUFFIX binding into $DAVEY_SCOPE_DIR" && \
      cd / && rm -rf /tmp/davey-fix; \
    fi

# Switch to non-root user
USER node

# QMD is installed on-demand at runtime when enabled in Settings.
ENV NODE_PATH="/data/.bun/install/global/node_modules:/home/node/.bun/install/global/node_modules"

# Health check
# Use the shallow gateway probe endpoints that upstream exposes specifically
# for container/runtime liveness. Routes like /__openclaw__/canvas/ now carry
# auth/capability semantics and can fail during normal startup even when the
# gateway process itself is healthy enough to finish booting.
HEALTHCHECK --interval=10s --timeout=3s --start-period=5s \
    CMD sh -c 'curl -fsS "http://localhost:${OPENCLAW_GATEWAY_PORT:-18789}/healthz" >/dev/null || exit 1'

EXPOSE 18789 19791 19793

# Set credentials directory to persistent storage
ENV OPENCLAW_OAUTH_DIR=/data/credentials

# Entrypoint script that creates auth from env vars, then starts gateway
COPY --chown=node:node entrypoint.sh /app/entrypoint.sh
RUN chmod +x /app/entrypoint.sh

ENTRYPOINT ["/app/entrypoint.sh"]
