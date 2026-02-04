import React, { useEffect, useMemo, useRef, useState } from "react";

/**
 * Map Marker Connector (working baseline + quick save)
 *
 * Features:
 * - Upload a map image.
 * - Add up to 50 CONNECTING markers.
 * - Add NON-CONNECTING waypoints: City, Waystation, Prepper, Distribution Centre.
 * - Connections: markers connect if distance <= min(thrA, thrB) where thr is 300m or 350m.
 * - Per-marker threshold editable when selected.
 * - Max 4 connections per marker (greedy shortest edges).
 * - Marker color by degree: 0 = hollow, 1–2 = green, 3–4 = orange.
 * - 350m markers show an extra white outline.
 * - Drag-move items only when hovered.
 * - Hide menu button for precise placement.
 * - Scale presets: 1m=1px (mPerPx=1) or 1m=10px (mPerPx=0.1).
 *
 * Quick Save:
 * - Auto-saves (debounced) map (data URL) + overlays + key settings to localStorage.
 * - On next open, auto-restores last session.
 * - Manual buttons: Save now / Restore / Clear saved.
 */

/* eslint-disable react-hooks/exhaustive-deps */

// Optional: if you deploy a default map at /map4096.png, the app will auto-load it
// when there is no saved session and no user-uploaded map yet.

const clamp = (n: number, a: number, b: number) => Math.max(a, Math.min(b, n));

type MarkerThreshold = 300 | 350;

type Marker = {
  id: string;
  x: number; // image px
  y: number; // image px
  thr: MarkerThreshold;
};

type WaypointType = "City" | "Waystation" | "Prepper" | "Distribution Centre";

type Waypoint = {
  id: string;
  x: number; // image px
  y: number; // image px
  type: WaypointType;
};

type Graph = { edges: Array<[number, number]>; degree: number[] };

type Selected =
  | { kind: "marker"; id: string }
  | { kind: "waypoint"; id: string }
  | null;

type ToolMode = "marker" | WaypointType;

type SessionV1 = {
  v: 1;
  savedAt: number;
  imgName: string;
  imgDataUrl: string;
  markers: Marker[];
  waypoints: Waypoint[];
  settings: {
    tool: ToolMode;
    newMarkerThr: MarkerThreshold;
    mPerPx: number;
    lockScale: boolean;
    sidebarHidden: boolean;
    showLines: boolean;
    lineAlpha: number;
    ringRadiusPx: number;
    ringStrokePx: number;
  };
};

const SESSION_KEY = "map_marker_connector_session_v1";

function uuid() {
  return typeof crypto !== "undefined" && (crypto as any).randomUUID
    ? (crypto as any).randomUUID()
    : String(Date.now()) + Math.random();
}

function markerColorForDegree(deg: number) {
  if (deg >= 3) return { stroke: "#ff9f1a", fill: "rgba(255, 159, 26, 0.78)" };
  return { stroke: "#19d319", fill: "rgba(25, 211, 25, 0.70)" };
}

function computeGraph(markers: Marker[], mPerPx: number, maxDegree: number): Graph {
  const edges: Array<[number, number]> = [];
  const degree = new Array(markers.length).fill(0);

  if (!Number.isFinite(mPerPx) || mPerPx <= 0) return { edges, degree };
  const cap = Math.max(0, Math.floor(maxDegree));

  const candidates: Array<{ i: number; j: number; d: number }> = [];
  for (let i = 0; i < markers.length; i++) {
    for (let j = i + 1; j < markers.length; j++) {
      const dx = markers[i].x - markers[j].x;
      const dy = markers[i].y - markers[j].y;
      const d = Math.hypot(dx, dy);
      const thrM = Math.min(markers[i].thr, markers[j].thr);
      const thrPx = thrM / mPerPx;
      if (d <= thrPx) candidates.push({ i, j, d });
    }
  }

  candidates.sort((a, b) => a.d - b.d);
  for (const c of candidates) {
    if (degree[c.i] >= cap || degree[c.j] >= cap) continue;
    edges.push([c.i, c.j]);
    degree[c.i] += 1;
    degree[c.j] += 1;
  }

  return { edges, degree };
}

function safeParseSession(raw: string | null): SessionV1 | null {
  if (!raw) return null;
  try {
    const obj = JSON.parse(raw);
    if (!obj || typeof obj !== "object") return null;
    if (obj.v !== 1) return null;
    if (typeof obj.imgDataUrl !== "string" || obj.imgDataUrl.length < 16) return null;
    if (!Array.isArray(obj.markers) || !Array.isArray(obj.waypoints)) return null;
    return obj as SessionV1;
  } catch {
    return null;
  }
}

function runSelfTests() {
  console.assert(clamp(5, 0, 10) === 5, "clamp basic");
  console.assert(clamp(-1, 0, 10) === 0, "clamp low");
  console.assert(clamp(99, 0, 10) === 10, "clamp high");

  const markers: Marker[] = [
    { id: "a", x: 0, y: 0, thr: 300 },
    { id: "b", x: 0, y: 299, thr: 300 },
    { id: "c", x: 0, y: 349, thr: 350 },
  ];
  const g1 = computeGraph(markers, 1, 4);
  console.assert(g1.edges.some(([i, j]) => (i === 0 && j === 1) || (i === 1 && j === 0)), "a-b connect under 300");
  console.assert(!g1.edges.some(([i, j]) => (i === 0 && j === 2) || (i === 2 && j === 0)), "a-c not connect (min thr) ");

  const s: SessionV1 = {
    v: 1,
    savedAt: Date.now(),
    imgName: "x.png",
    imgDataUrl: "data:image/png;base64,AAAA",
    markers: [],
    waypoints: [],
    settings: {
      tool: "marker",
      newMarkerThr: 300,
      mPerPx: 1,
      lockScale: true,
      sidebarHidden: false,
      showLines: true,
      lineAlpha: 0.3,
      ringRadiusPx: 14,
      ringStrokePx: 3,
    },
  };
  console.assert(!!safeParseSession(JSON.stringify(s)), "safeParseSession accepts valid v1");
  console.assert(safeParseSession("{bad") === null, "safeParseSession rejects invalid json");
}

export default function App() {
  const containerRef = useRef<HTMLDivElement | null>(null);
  const canvasRef = useRef<HTMLCanvasElement | null>(null);

  const [img, setImg] = useState<HTMLImageElement | null>(null);
  const [imgName, setImgName] = useState<string>("No map loaded");
  const [imgDataUrl, setImgDataUrl] = useState<string>("");
  const [imgStatus, setImgStatus] = useState<"idle" | "loading" | "loaded" | "error">("idle");
  const [imgError, setImgError] = useState<string>("");

  const [markers, setMarkers] = useState<Marker[]>([]);
  const [waypoints, setWaypoints] = useState<Waypoint[]>([]);

  const [tool, setTool] = useState<ToolMode>("marker");
  const [newMarkerThr, setNewMarkerThr] = useState<MarkerThreshold>(300);

  const [mPerPx, setMPerPx] = useState<number>(1);
  const [lockScale, setLockScale] = useState<boolean>(true);

  const [sidebarHidden, setSidebarHidden] = useState<boolean>(false);

  const [showLines, setShowLines] = useState<boolean>(true);
  const [lineAlpha, setLineAlpha] = useState<number>(0.35);
  const [ringRadiusPx, setRingRadiusPx] = useState<number>(14);
  const [ringStrokePx, setRingStrokePx] = useState<number>(3);

  const [selected, setSelected] = useState<Selected>(null);
  const [hovered, setHovered] = useState<Selected>(null);
  const [dragging, setDragging] = useState<Selected>(null);

  const dragOffsetRef = useRef<{ dx: number; dy: number } | null>(null);
  const justDraggedRef = useRef<boolean>(false);

  const [saveMsg, setSaveMsg] = useState<string>("");
  const saveTimerRef = useRef<number | null>(null);

  useEffect(() => {
    runSelfTests();
  }, []);

  const { edges, degree } = useMemo(() => computeGraph(markers, mPerPx, 4), [markers, mPerPx]);

  const selectedMarkerIndex = useMemo(() => {
    if (!selected || selected.kind !== "marker") return -1;
    return markers.findIndex((m) => m.id === selected.id);
  }, [selected, markers]);

  const selectedMarker = selectedMarkerIndex >= 0 ? markers[selectedMarkerIndex] : null;

  const selectedWaypointIndex = useMemo(() => {
    if (!selected || selected.kind !== "waypoint") return -1;
    return waypoints.findIndex((w) => w.id === selected.id);
  }, [selected, waypoints]);

  const selectedWaypoint = selectedWaypointIndex >= 0 ? waypoints[selectedWaypointIndex] : null;

  const loadImageFromDataUrl = (name: string, dataUrl: string, afterLoad?: () => void) => {
    setImgName(name || "(restored)");
    setImgStatus("loading");
    setImgError("");

    const im = new Image();
    im.onload = async () => {
      try {
        if (typeof (im as any).decode === "function") await (im as any).decode();
      } catch {
        // ignore
      }
      setImg(im);
      setImgDataUrl(dataUrl);
      setImgStatus("loaded");
      afterLoad?.();
    };
    im.onerror = () => {
      setImg(null);
      setImgStatus("error");
      setImgError("Failed to decode the image.");
    };
    im.src = dataUrl;
  };

  const handleLoadImage = (file: File | undefined) => {
    if (!file) return;

    setImgName(file.name || "(unnamed)");
    setImgStatus("loading");
    setImgError("");

    const reader = new FileReader();
    reader.onerror = () => {
      setImg(null);
      setImgStatus("error");
      setImgError("Failed to read the selected file.");
    };
    reader.onload = () => {
      const dataUrl = reader.result;
      if (typeof dataUrl !== "string") {
        setImg(null);
        setImgStatus("error");
        setImgError("Unexpected file reader result.");
        return;
      }

      loadImageFromDataUrl(file.name || "(unnamed)", dataUrl, () => {
        setMarkers([]);
        setWaypoints([]);
        setSelected(null);
        setHovered(null);
        setDragging(null);
        setTool("marker");
        setNewMarkerThr(300);
        setMPerPx(1);
        setLockScale(true);
      });
    };
    reader.readAsDataURL(file);
  };

  const persistNow = () => {
    if (!imgDataUrl || imgStatus !== "loaded") {
      setSaveMsg("Nothing to save yet (no map loaded).");
      return;
    }

    const session: SessionV1 = {
      v: 1,
      savedAt: Date.now(),
      imgName,
      imgDataUrl,
      markers,
      waypoints,
      settings: {
        tool,
        newMarkerThr,
        mPerPx,
        lockScale,
        sidebarHidden,
        showLines,
        lineAlpha,
        ringRadiusPx,
        ringStrokePx,
      },
    };

    try {
      localStorage.setItem(SESSION_KEY, JSON.stringify(session));
      setSaveMsg(`Saved ✓ (${new Date(session.savedAt).toLocaleString()})`);
    } catch {
      setSaveMsg("Save failed (localStorage quota likely exceeded).");
    }
  };

  const restoreLast = () => {
    const session = safeParseSession(localStorage.getItem(SESSION_KEY));
    if (!session) {
      setSaveMsg("No saved session found.");
      return;
    }

    loadImageFromDataUrl(session.imgName, session.imgDataUrl, () => {
      setMarkers(session.markers || []);
      setWaypoints(session.waypoints || []);
      setSelected(null);
      setHovered(null);
      setDragging(null);

      setTool(session.settings?.tool ?? "marker");
      setNewMarkerThr((session.settings?.newMarkerThr as MarkerThreshold) ?? 300);
      setMPerPx(Number(session.settings?.mPerPx) || 1);
      setLockScale(!!session.settings?.lockScale);
      setSidebarHidden(!!session.settings?.sidebarHidden);
      setShowLines(session.settings?.showLines ?? true);
      setLineAlpha(Number(session.settings?.lineAlpha) || 0.35);
      setRingRadiusPx(Number(session.settings?.ringRadiusPx) || 14);
      setRingStrokePx(Number(session.settings?.ringStrokePx) || 3);

      setSaveMsg(`Restored ✓ (${new Date(session.savedAt).toLocaleString()})`);
    });
  };

  const clearSaved = () => {
    try {
      localStorage.removeItem(SESSION_KEY);
      setSaveMsg("Saved session cleared.");
    } catch {
      setSaveMsg("Failed to clear saved session.");
    }
  };

  // Auto-restore on first mount (preferred)
  useEffect(() => {
    const session = safeParseSession(localStorage.getItem(SESSION_KEY));
    if (!session) return;
    if (img) return;
    restoreLast();
  }, []);

  // If no saved session, try auto-load a default map from /map4096.png (optional)
  useEffect(() => {
    const hasSession = !!safeParseSession(localStorage.getItem(SESSION_KEY));
    if (hasSession) return;
    if (img) return;

    (async () => {
      try {
        const res = await fetch("/map4096.png", { cache: "no-cache" });
        if (!res.ok) return;
        const blob = await res.blob();
        const reader = new FileReader();
        reader.onload = () => {
          const dataUrl = reader.result;
          if (typeof dataUrl !== "string") return;
          loadImageFromDataUrl("map4096.png", dataUrl);
          // base-map defaults
          setMPerPx(1);
          setLockScale(true);
        };
        reader.readAsDataURL(blob);
      } catch {
        // ignore
      }
    })();
  }, [img]);

  // Auto-save (debounced)
  useEffect(() => {
    if (!imgDataUrl || imgStatus !== "loaded") return;

    if (saveTimerRef.current) window.clearTimeout(saveTimerRef.current);
    saveTimerRef.current = window.setTimeout(() => {
      try {
        const session: SessionV1 = {
          v: 1,
          savedAt: Date.now(),
          imgName,
          imgDataUrl,
          markers,
          waypoints,
          settings: {
            tool,
            newMarkerThr,
            mPerPx,
            lockScale,
            sidebarHidden,
            showLines,
            lineAlpha,
            ringRadiusPx,
            ringStrokePx,
          },
        };
        localStorage.setItem(SESSION_KEY, JSON.stringify(session));
        setSaveMsg(`Auto-saved ✓ (${new Date(session.savedAt).toLocaleTimeString()})`);
      } catch {
        setSaveMsg("Auto-save failed (localStorage quota likely exceeded).");
      }
    }, 600);

    return () => {
      if (saveTimerRef.current) window.clearTimeout(saveTimerRef.current);
    };
  }, [
    imgStatus,
    imgDataUrl,
    imgName,
    markers,
    waypoints,
    tool,
    newMarkerThr,
    mPerPx,
    lockScale,
    sidebarHidden,
    showLines,
    lineAlpha,
    ringRadiusPx,
    ringStrokePx,
  ]);

  const getDrawTransform = () => {
    const container = containerRef.current;
    if (!container || !img) return null;
    const rect = container.getBoundingClientRect();

    const cw = rect.width;
    const ch = rect.height;
    const iw = img.naturalWidth;
    const ih = img.naturalHeight;
    if (!iw || !ih) return null;

    const scale = Math.min(cw / iw, ch / ih);
    const dw = iw * scale;
    const dh = ih * scale;
    const ox = (cw - dw) / 2;
    const oy = (ch - dh) / 2;

    return { rect, cw, ch, iw, ih, scale, ox, oy, dw, dh };
  };

  const screenToImage = (sx: number, sy: number) => {
    const t = getDrawTransform();
    if (!t) return null;

    const x = sx - t.rect.left;
    const y = sy - t.rect.top;

    if (x < t.ox || x > t.ox + t.dw || y < t.oy || y > t.oy + t.dh) return null;

    return {
      x: (x - t.ox) / t.scale,
      y: (y - t.oy) / t.scale,
      iw: t.iw,
      ih: t.ih,
    };
  };

  const hitTestAny = (sx: number, sy: number): Selected => {
    const t = getDrawTransform();
    if (!t) return null;

    const x = sx - t.rect.left;
    const y = sy - t.rect.top;

    if (x < t.ox || x > t.ox + t.dw || y < t.oy || y > t.oy + t.dh) return null;

    const r = ringRadiusPx * 1.25;

    // Markers first
    for (let i = markers.length - 1; i >= 0; i--) {
      const mx = t.ox + markers[i].x * t.scale;
      const my = t.oy + markers[i].y * t.scale;
      if (Math.hypot(x - mx, y - my) <= r) return { kind: "marker", id: markers[i].id };
    }

    // Waypoints
    for (let i = waypoints.length - 1; i >= 0; i--) {
      const wx = t.ox + waypoints[i].x * t.scale;
      const wy = t.oy + waypoints[i].y * t.scale;
      if (Math.hypot(x - wx, y - wy) <= r) return { kind: "waypoint", id: waypoints[i].id };
    }

    return null;
  };

  const onMouseMoveCanvas = (e: React.MouseEvent<HTMLCanvasElement>) => {
    if (!img) return;

    if (dragging) {
      const p = screenToImage(e.clientX, e.clientY);
      if (!p) return;
      const off = dragOffsetRef.current;
      if (!off) return;

      const nx = clamp(p.x - off.dx, 0, p.iw);
      const ny = clamp(p.y - off.dy, 0, p.ih);

      if (dragging.kind === "marker") {
        setMarkers((prev) => prev.map((m) => (m.id === dragging.id ? { ...m, x: nx, y: ny } : m)));
      } else {
        setWaypoints((prev) => prev.map((w) => (w.id === dragging.id ? { ...w, x: nx, y: ny } : w)));
      }

      justDraggedRef.current = true;
      return;
    }

    setHovered(hitTestAny(e.clientX, e.clientY));
  };

  const onMouseDownCanvas = (e: React.MouseEvent<HTMLCanvasElement>) => {
    if (!img) return;
    if (!hovered) return;

    const p = screenToImage(e.clientX, e.clientY);
    if (!p) return;

    if (hovered.kind === "marker") {
      const m = markers.find((mm) => mm.id === hovered.id);
      if (!m) return;
      dragOffsetRef.current = { dx: p.x - m.x, dy: p.y - m.y };
    } else {
      const w = waypoints.find((ww) => ww.id === hovered.id);
      if (!w) return;
      dragOffsetRef.current = { dx: p.x - w.x, dy: p.y - w.y };
    }

    setDragging(hovered);
    setSelected(hovered);
    justDraggedRef.current = false;
  };

  const stopDragging = () => {
    setDragging(null);
    dragOffsetRef.current = null;
  };

  const onMouseUpCanvas = () => {
    if (!dragging) return;
    stopDragging();
  };

  const onMouseLeaveCanvas = () => {
    if (!dragging) setHovered(null);
  };

  const onClickCanvas = (e: React.MouseEvent<HTMLCanvasElement>) => {
    if (!img) return;

    if (justDraggedRef.current) {
      justDraggedRef.current = false;
      return;
    }

    const hit = hitTestAny(e.clientX, e.clientY);
    if (hit) {
      setSelected(hit);
      return;
    }

    const p = screenToImage(e.clientX, e.clientY);
    if (!p) return;

    if (tool === "marker") {
      if (markers.length >= 50) return;
      const id = uuid();
      setMarkers((prev) => [...prev, { id, x: p.x, y: p.y, thr: newMarkerThr }]);
      setSelected({ kind: "marker", id });
    } else {
      const id = uuid();
      setWaypoints((prev) => [...prev, { id, x: p.x, y: p.y, type: tool }]);
      setSelected({ kind: "waypoint", id });
    }
  };

  const setSelectedThreshold = (thr: MarkerThreshold) => {
    if (!selected || selected.kind !== "marker") return;
    setMarkers((prev) => prev.map((m) => (m.id === selected.id ? { ...m, thr } : m)));
  };

  const undoSmart = () => {
    setSelected(null);
    setHovered(null);
    stopDragging();

    setMarkers((mPrev) => {
      if (mPrev.length > 0) return mPrev.slice(0, -1);
      return mPrev;
    });

    // If there are no markers, undo a waypoint instead.
    setTimeout(() => {
      setMarkers((mNow) => {
        if (mNow.length === 0) {
          setWaypoints((wNow) => (wNow.length > 0 ? wNow.slice(0, -1) : wNow));
        }
        return mNow;
      });
    }, 0);
  };

  const clearOverlays = () => {
    setMarkers([]);
    setWaypoints([]);
    setSelected(null);
    setHovered(null);
    stopDragging();
  };

  // Cursor feedback
  useEffect(() => {
    const canvas = canvasRef.current;
    if (!canvas) return;

    if (!img) {
      canvas.style.cursor = "default";
      return;
    }

    if (dragging) canvas.style.cursor = "grabbing";
    else if (hovered) canvas.style.cursor = "grab";
    else canvas.style.cursor = tool === "marker" ? "crosshair" : "copy";
  }, [hovered, dragging, img, tool]);

  // Draw
  useEffect(() => {
    const canvas = canvasRef.current;
    const container = containerRef.current;
    if (!canvas || !container) return;

    const ctx = canvas.getContext("2d");
    if (!ctx) return;

    const dpr = window.devicePixelRatio || 1;

    const redraw = () => {
      const rect = container.getBoundingClientRect();
      const cw = rect.width;
      const ch = rect.height;

      canvas.style.width = `${cw}px`;
      canvas.style.height = `${ch}px`;
      canvas.width = Math.floor(cw * dpr);
      canvas.height = Math.floor(ch * dpr);

      ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
      ctx.clearRect(0, 0, cw, ch);

      if (!img) {
        ctx.fillStyle = "#0b0f14";
        ctx.fillRect(0, 0, cw, ch);
        ctx.fillStyle = "rgba(255,255,255,0.10)";
        ctx.font = "600 14px ui-sans-serif, system-ui, -apple-system, Segoe UI, Roboto";
        ctx.fillText("Load the base map PNG via the file picker or Restore.", 16, 28);
        ctx.font = "12px ui-sans-serif, system-ui, -apple-system, Segoe UI, Roboto";
        ctx.fillText("Tool: Marker / City / Waystation / Prepper / Distribution Centre.", 16, 48);
        return;
      }

      const iw = img.naturalWidth;
      const ih = img.naturalHeight;
      const scale = Math.min(cw / iw, ch / ih);
      const dw = iw * scale;
      const dh = ih * scale;
      const ox = (cw - dw) / 2;
      const oy = (ch - dh) / 2;

      ctx.imageSmoothingEnabled = true;
      ctx.imageSmoothingQuality = "high";
      ctx.drawImage(img, ox, oy, dw, dh);

      const toScreen = (p: { x: number; y: number }) => ({
        x: ox + p.x * scale,
        y: oy + p.y * scale,
      });

      // Lines
      if (showLines && edges.length > 0) {
        ctx.save();
        ctx.globalAlpha = clamp(lineAlpha, 0, 1);
        ctx.strokeStyle = "#69ff6b";
        ctx.lineWidth = 2;
        ctx.beginPath();
        for (const [i, j] of edges) {
          const a = toScreen(markers[i]);
          const b = toScreen(markers[j]);
          ctx.moveTo(a.x, a.y);
          ctx.lineTo(b.x, b.y);
        }
        ctx.stroke();
        ctx.restore();
      }

      // Waypoints
      for (let i = 0; i < waypoints.length; i++) {
        const p = toScreen(waypoints[i]);
        const isSelected = selected?.kind === "waypoint" && selected.id === waypoints[i].id;
        const isHover = hovered?.kind === "waypoint" && hovered.id === waypoints[i].id;

        ctx.save();

        const scaleFactor =
          waypoints[i].type === "City"
            ? 1
            : waypoints[i].type === "Waystation"
            ? 0.5
            : waypoints[i].type === "Prepper"
            ? 0.75
            : 0.67;

        const r = ringRadiusPx * scaleFactor;
        const x = p.x - r;
        const y = p.y - r;

        if (isSelected || isHover) {
          ctx.globalAlpha = isSelected ? 0.9 : 0.65;
          ctx.strokeStyle = "rgba(255,255,255,0.7)";
          ctx.lineWidth = 2;
          ctx.beginPath();
          ctx.roundRect(x - 5, y - 5, r * 2 + 10, r * 2 + 10, 10);
          ctx.stroke();
        }

        ctx.globalAlpha = 1;
        ctx.fillStyle = "rgba(30, 120, 255, 0.55)";
        ctx.strokeStyle = "rgba(255,255,255,0.85)";
        ctx.lineWidth = 2;
        ctx.beginPath();
        ctx.roundRect(x, y, r * 2, r * 2, 10);
        ctx.fill();
        ctx.stroke();

        const letter =
          waypoints[i].type === "City"
            ? "C"
            : waypoints[i].type === "Waystation"
            ? "W"
            : waypoints[i].type === "Prepper"
            ? "P"
            : "D";

        ctx.fillStyle = "rgba(0,0,0,0.9)";
        ctx.font = "800 12px ui-sans-serif, system-ui, -apple-system, Segoe UI, Roboto";
        ctx.textAlign = "center";
        ctx.textBaseline = "middle";
        ctx.fillText(letter, p.x, p.y);

        ctx.fillStyle = "rgba(255,255,255,0.85)";
        ctx.font = "700 10px ui-sans-serif, system-ui, -apple-system, Segoe UI, Roboto";
        ctx.fillText(waypoints[i].type, p.x, p.y + r + 12);

        ctx.restore();
      }

      // Markers
      for (let i = 0; i < markers.length; i++) {
        const p = toScreen(markers[i]);
        const deg = degree[i] || 0;
        const isConnected = deg > 0;
        const isSelected = selected?.kind === "marker" && selected.id === markers[i].id;
        const isHover = hovered?.kind === "marker" && hovered.id === markers[i].id;

        ctx.save();

        if (isSelected) {
          ctx.globalAlpha = 0.9;
          ctx.strokeStyle = "rgba(255,255,255,0.75)";
          ctx.lineWidth = 2;
          ctx.beginPath();
          ctx.arc(p.x, p.y, ringRadiusPx + 7, 0, Math.PI * 2);
          ctx.stroke();
        }

        if (isHover && !isSelected) {
          ctx.globalAlpha = 0.6;
          ctx.strokeStyle = "rgba(255,255,255,0.55)";
          ctx.lineWidth = 2;
          ctx.beginPath();
          ctx.arc(p.x, p.y, ringRadiusPx + 6, 0, Math.PI * 2);
          ctx.stroke();
        }

        ctx.globalAlpha = 1;
        ctx.lineWidth = ringStrokePx;

        if (isConnected) {
          const col = markerColorForDegree(deg);
          ctx.strokeStyle = col.stroke;
          ctx.fillStyle = col.fill;
        } else {
          ctx.strokeStyle = "rgba(255,255,255,0.85)";
          ctx.fillStyle = "rgba(0,0,0,0)";
        }

        ctx.beginPath();
        ctx.arc(p.x, p.y, ringRadiusPx, 0, Math.PI * 2);
        ctx.fill();
        ctx.stroke();

        if (markers[i].thr === 350) {
          ctx.save();
          ctx.globalAlpha = 0.95;
          ctx.strokeStyle = "rgba(255,255,255,0.9)";
          ctx.lineWidth = 2;
          ctx.beginPath();
          ctx.arc(p.x, p.y, ringRadiusPx + 2.5, 0, Math.PI * 2);
          ctx.stroke();
          ctx.restore();
        }

        if (!isConnected) {
          ctx.globalCompositeOperation = "destination-out";
          ctx.beginPath();
          ctx.arc(p.x, p.y, Math.max(1, ringRadiusPx - ringStrokePx - 1), 0, Math.PI * 2);
          ctx.fill();
          ctx.globalCompositeOperation = "source-over";
        }

        ctx.fillStyle = isConnected ? "rgba(0,0,0,0.9)" : "rgba(255,255,255,0.9)";
        ctx.font = "600 11px ui-sans-serif, system-ui, -apple-system, Segoe UI, Roboto";
        ctx.textAlign = "center";
        ctx.textBaseline = "middle";
        ctx.fillText(String(i + 1), p.x, p.y);

        if (isConnected) {
          ctx.fillStyle = "rgba(0,0,0,0.85)";
          ctx.font = "800 9px ui-sans-serif, system-ui, -apple-system, Segoe UI, Roboto";
          ctx.fillText(String(deg), p.x, p.y + ringRadiusPx * 0.72);
        }

        ctx.fillStyle = "rgba(255,255,255,0.75)";
        ctx.font = "700 9px ui-sans-serif, system-ui, -apple-system, Segoe UI, Roboto";
        ctx.fillText(`${markers[i].thr}`, p.x, p.y - ringRadiusPx * 0.9);

        ctx.restore();
      }

      // HUD
      ctx.save();
      ctx.fillStyle = "rgba(0,0,0,0.55)";
      ctx.fillRect(12, ch - 46, 820, 34);
      ctx.fillStyle = "rgba(255,255,255,0.9)";
      ctx.font = "12px ui-sans-serif, system-ui, -apple-system, Segoe UI, Roboto";
      ctx.fillText(
        `Markers: ${markers.length}/50   Waypoints: ${waypoints.length}   Edges: ${edges.length}   Tool: ${tool}   Scale: ${mPerPx} m/px`,
        22,
        ch - 25
      );
      ctx.restore();
    };

    redraw();
    const ro = new ResizeObserver(() => redraw());
    ro.observe(container);
    return () => ro.disconnect();
  }, [img, markers, waypoints, edges, degree, showLines, lineAlpha, ringRadiusPx, ringStrokePx, selected, hovered, mPerPx, tool]);

  return (
    <div className="min-h-screen w-full bg-zinc-950 text-zinc-100 p-3">
      <div className="mx-auto max-w-7xl">
        <div className="flex items-center justify-between gap-2 mb-3">
          <div className="text-sm text-zinc-200">
            <span className="font-semibold">Map Marker Connector</span>
            <span className="text-zinc-400"> · hover+drag to move</span>
            {saveMsg ? <span className="ml-3 text-xs text-zinc-400">{saveMsg}</span> : null}
          </div>
          <div className="flex items-center gap-2">
            <button onClick={() => setSidebarHidden((v) => !v)} className="rounded-xl bg-zinc-800 hover:bg-zinc-700 px-3 py-2 text-sm">
              {sidebarHidden ? "Show menu" : "Hide menu"}
            </button>
          </div>
        </div>

        <div className={`grid gap-3 ${sidebarHidden ? "grid-cols-1" : "grid-cols-1 lg:grid-cols-[360px_1fr]"}`}>
          {!sidebarHidden ? (
            <div className="rounded-2xl bg-zinc-900/50 border border-zinc-800 shadow-xl p-4">
              <div className="space-y-3">
                <div>
                  <div className="text-sm font-semibold">Map</div>
                  <div className="text-xs text-zinc-400 mt-1">
                    {img ? (
                      <span>
                        Loaded: <span className="text-zinc-200">{imgName}</span> ({img.naturalWidth}×{img.naturalHeight})
                      </span>
                    ) : imgStatus === "loading" ? (
                      <span>Loading…</span>
                    ) : imgStatus === "error" ? (
                      <span className="text-red-300">{imgError}</span>
                    ) : (
                      <span>No image loaded.</span>
                    )}
                  </div>
                  <input
                    type="file"
                    accept="image/*"
                    onChange={(e) => handleLoadImage(e.target.files?.[0])}
                    className="mt-2 block w-full text-sm file:mr-3 file:rounded-xl file:border-0 file:bg-zinc-800 file:px-3 file:py-2 file:text-zinc-100 hover:file:bg-zinc-700"
                  />
                </div>

                <div className="rounded-xl border border-zinc-800 bg-zinc-950/30 p-3 space-y-2">
                  <div className="text-sm font-semibold">Quick save</div>
                  <div className="flex flex-wrap gap-2">
                    <button onClick={persistNow} className="rounded-xl bg-zinc-800 hover:bg-zinc-700 px-3 py-2 text-sm">
                      Save now
                    </button>
                    <button onClick={restoreLast} className="rounded-xl bg-zinc-800 hover:bg-zinc-700 px-3 py-2 text-sm">
                      Restore
                    </button>
                    <button onClick={clearSaved} className="rounded-xl bg-zinc-800 hover:bg-zinc-700 px-3 py-2 text-sm">
                      Clear saved
                    </button>
                  </div>
                  <div className="text-xs text-zinc-400">Auto-save is enabled. Map image is stored locally in your browser.</div>
                </div>

                <div className="rounded-xl border border-zinc-800 bg-zinc-950/30 p-3 space-y-2">
                  <div className="text-sm font-semibold">Tool</div>
                  <div className="grid grid-cols-2 gap-2 text-sm">
                    <label className="flex items-center gap-2">
                      <input type="radio" checked={tool === "marker"} onChange={() => setTool("marker")} />
                      Marker
                    </label>
                    <label className="flex items-center gap-2">
                      <input type="radio" checked={tool === "City"} onChange={() => setTool("City")} />
                      City
                    </label>
                    <label className="flex items-center gap-2">
                      <input type="radio" checked={tool === "Waystation"} onChange={() => setTool("Waystation")} />
                      Waystation
                    </label>
                    <label className="flex items-center gap-2">
                      <input type="radio" checked={tool === "Prepper"} onChange={() => setTool("Prepper")} />
                      Prepper
                    </label>
                    <label className="flex items-center gap-2">
                      <input type="radio" checked={tool === "Distribution Centre"} onChange={() => setTool("Distribution Centre")} />
                      Distribution
                    </label>
                  </div>
                  <div className="text-xs text-zinc-400">Markers connect. Waypoints do not.</div>
                </div>

                <div className={`rounded-xl border border-zinc-800 bg-zinc-950/30 p-3 space-y-2 ${tool !== "marker" ? "opacity-50" : ""}`}>
                  <div className="flex items-center justify-between">
                    <span className="text-sm">New marker threshold</span>
                    <span className="text-xs text-zinc-400">{newMarkerThr}m</span>
                  </div>
                  <div className="flex items-center gap-2">
                    <label className="flex items-center gap-2 text-sm">
                      <input type="radio" disabled={tool !== "marker"} checked={newMarkerThr === 300} onChange={() => setNewMarkerThr(300)} />
                      300
                    </label>
                    <label className="flex items-center gap-2 text-sm">
                      <input type="radio" disabled={tool !== "marker"} checked={newMarkerThr === 350} onChange={() => setNewMarkerThr(350)} />
                      350
                    </label>
                  </div>
                </div>

                <div className="rounded-xl border border-zinc-800 bg-zinc-950/30 p-3 space-y-2">
                  <div className="text-sm font-semibold">Selected</div>
                  {selectedMarker ? (
                    <>
                      <div className="text-xs text-zinc-400">
                        Marker #{selectedMarkerIndex + 1} · threshold: <span className="text-zinc-200">{selectedMarker.thr}m</span>
                      </div>
                      <div className="flex items-center gap-2">
                        <button
                          onClick={() => setSelectedThreshold(300)}
                          className={`rounded-xl px-3 py-2 text-sm border ${selectedMarker.thr === 300 ? "bg-zinc-800 border-zinc-600" : "bg-zinc-900/40 border-zinc-800 hover:bg-zinc-800"}`}
                        >
                          300m
                        </button>
                        <button
                          onClick={() => setSelectedThreshold(350)}
                          className={`rounded-xl px-3 py-2 text-sm border ${selectedMarker.thr === 350 ? "bg-zinc-800 border-zinc-600" : "bg-zinc-900/40 border-zinc-800 hover:bg-zinc-800"}`}
                        >
                          350m
                        </button>
                      </div>
                    </>
                  ) : selectedWaypoint ? (
                    <div className="text-xs text-zinc-400">
                      Waypoint · <span className="text-zinc-200">{selectedWaypoint.type}</span>
                    </div>
                  ) : (
                    <div className="text-xs text-zinc-400">Click a marker/waypoint to select it.</div>
                  )}
                </div>

                <div className="rounded-xl border border-zinc-800 bg-zinc-950/30 p-3 space-y-3">
                  <div className="flex items-center justify-between">
                    <span className="text-sm">Scale (meters ↔ pixels)</span>
                    <label className="flex items-center gap-2 text-xs text-zinc-400">
                      <input type="checkbox" checked={lockScale} onChange={(e) => setLockScale(e.target.checked)} />
                      Lock
                    </label>
                  </div>
                  <div className="flex items-center gap-2">
                    <button
                      disabled={lockScale}
                      onClick={() => setMPerPx(1)}
                      className={`px-3 py-2 rounded-xl text-sm border ${mPerPx === 1 ? "bg-zinc-800 border-zinc-600" : "bg-zinc-900/40 border-zinc-800 hover:bg-zinc-800"} ${lockScale ? "opacity-40" : ""}`}
                    >
                      1 m = 1 px
                    </button>
                    <button
                      disabled={lockScale}
                      onClick={() => setMPerPx(0.1)}
                      className={`px-3 py-2 rounded-xl text-sm border ${mPerPx === 0.1 ? "bg-zinc-800 border-zinc-600" : "bg-zinc-900/40 border-zinc-800 hover:bg-zinc-800"} ${lockScale ? "opacity-40" : ""}`}
                    >
                      1 m = 10 px
                    </button>
                  </div>
                </div>

                <div className="grid grid-cols-2 gap-3">
                  <div className="space-y-2">
                    <label className="text-sm font-medium">Marker radius</label>
                    <input type="range" min={8} max={24} value={ringRadiusPx} onChange={(e) => setRingRadiusPx(Number(e.target.value))} className="w-full" />
                  </div>
                  <div className="space-y-2">
                    <label className="text-sm font-medium">Ring stroke</label>
                    <input type="range" min={2} max={6} value={ringStrokePx} onChange={(e) => setRingStrokePx(Number(e.target.value))} className="w-full" />
                  </div>
                </div>

                <div className="rounded-xl border border-zinc-800 bg-zinc-950/30 p-3 space-y-2">
                  <div className="flex items-center justify-between">
                    <span className="text-sm">Show lines</span>
                    <input type="checkbox" checked={showLines} onChange={(e) => setShowLines(e.target.checked)} />
                  </div>
                  <div className="flex items-center justify-between gap-3">
                    <span className="text-sm">Line opacity</span>
                    <input type="range" min={0.05} max={1} step={0.05} value={lineAlpha} onChange={(e) => setLineAlpha(Number(e.target.value))} className="w-full" />
                    <span className="text-sm tabular-nums w-12 text-right">{lineAlpha.toFixed(2)}</span>
                  </div>
                </div>

                <div className="flex flex-wrap gap-2">
                  <button
                    onClick={undoSmart}
                    disabled={markers.length === 0 && waypoints.length === 0}
                    className="rounded-xl bg-zinc-800 hover:bg-zinc-700 disabled:opacity-40 disabled:hover:bg-zinc-800 px-3 py-2 text-sm"
                  >
                    Undo
                  </button>
                  <button
                    onClick={clearOverlays}
                    disabled={markers.length === 0 && waypoints.length === 0}
                    className="rounded-xl bg-zinc-800 hover:bg-zinc-700 disabled:opacity-40 disabled:hover:bg-zinc-800 px-3 py-2 text-sm"
                  >
                    Clear overlays
                  </button>
                </div>

                <div className="text-xs text-zinc-400">Waypoints are labeled C / W / P / D.</div>
              </div>
            </div>
          ) : null}

          <div className="rounded-2xl bg-zinc-900/50 border border-zinc-800 shadow-xl overflow-hidden relative">
            {sidebarHidden ? (
              <div className="absolute top-3 left-3 z-10 flex flex-wrap gap-2">
                <button onClick={persistNow} className="rounded-xl bg-zinc-800/90 hover:bg-zinc-700/90 px-3 py-2 text-sm">
                  Save
                </button>
                <button onClick={restoreLast} className="rounded-xl bg-zinc-800/90 hover:bg-zinc-700/90 px-3 py-2 text-sm">
                  Restore
                </button>
                <button
                  onClick={undoSmart}
                  disabled={markers.length === 0 && waypoints.length === 0}
                  className="rounded-xl bg-zinc-800/90 hover:bg-zinc-700/90 disabled:opacity-40 px-3 py-2 text-sm"
                >
                  Undo
                </button>
                <button
                  onClick={clearOverlays}
                  disabled={markers.length === 0 && waypoints.length === 0}
                  className="rounded-xl bg-zinc-800/90 hover:bg-zinc-700/90 disabled:opacity-40 px-3 py-2 text-sm"
                >
                  Clear
                </button>
              </div>
            ) : null}

            <div className="px-4 py-3 border-b border-zinc-800 flex items-center justify-between">
              <div className="text-sm text-zinc-200">
                <span className="font-medium">Canvas</span>
                <span className="text-zinc-400"> · tool: {tool}</span>
              </div>
              <div className="text-xs text-zinc-400 tabular-nums">
                {markers.length}/50 markers · {waypoints.length} waypoints · {edges.length} edges
              </div>
            </div>

            <div ref={containerRef} className={`relative w-full ${sidebarHidden ? "h-[84vh]" : "h-[72vh]"} bg-zinc-950`}>
              <canvas
                ref={canvasRef}
                onMouseMove={onMouseMoveCanvas}
                onMouseDown={onMouseDownCanvas}
                onMouseUp={onMouseUpCanvas}
                onMouseLeave={onMouseLeaveCanvas}
                onClick={onClickCanvas}
                className="absolute inset-0"
              />
              {!img ? (
                <div className="absolute inset-0 flex items-center justify-center pointer-events-none">
                  <div className="rounded-2xl border border-zinc-800 bg-zinc-950/70 px-4 py-3 text-sm text-zinc-200 shadow">
                    Load the base map PNG to start, or use Restore.
                  </div>
                </div>
              ) : null}
            </div>
          </div>
        </div>
      </div>
    </div>
  );
}
