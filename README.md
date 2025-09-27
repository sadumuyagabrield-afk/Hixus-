# Hixus-import express from "express";
import { PrismaClient } from "@prisma/client";

const router = express.Router();
const prisma = new PrismaClient();

// Fetch unified inbox (chat + notifications)
router.get("/:userId", async (req, res) => {
  const { userId } = req.params;

  // Notifications
  const notifications = await prisma.notification.findMany({
    where: { userId },
    select: { id: true, title: true, message: true, createdAt: true },
  });

  // Chat messages (last per room)
  const messages = await prisma.message.findMany({
    where: { room: { members: { some: { userId } } } },
    orderBy: { createdAt: "desc" },
    take: 20,
    include: { sender: true, room: true },
  });

  const unified = [
    ...notifications.map((n) => ({
      id: n.id,
      type: "notification",
      title: n.title,
      message: n.message,
      createdAt: n.createdAt,
    })),
    ...messages.map((m) => ({
      id: m.id,
      type: "message",
      title: m.sender.name || "Unknown",
      message: m.content,// New chat messages are already emitted
// Also emit inbox events
socket.on("message", async ({ roomId, senderId, content }) => {
  const message = await prisma.message.create({
    data: { roomId, senderId, content },
    include: { sender: true },
  });

  io.to(roomId).emit("message", message);

  // Emit to inbox as well
  message.room.members.forEach((member) => {
    io.to(member.userId).emit("inbox", {
      id: message.id,
      type: "message",
      title: message.sender.name,
      message: message.content,
      createdAt: message.createdAt,
      roomId,
    });
  });
});

// For"use client";
import { useEffect, useState } from "react";
import { io } from "socket.io-client";

interface InboxItem {
  id: string;
  type: "notification" | "message";
  title: string;
  message: string;
  createdAt: string;
  roomId?: string;
}

export function useInbox(userId: string) {
  const [items, setItems] = useState<InboxItem[]>([]);

  useEffect(() => {
    const s = io(process.env.NEXT_PUBLIC_API_URL || "http://localhost:4000");

    s.emit("join", userId);

    s.on("inbox", (item: InboxItem) => {
      setItems((prev) => [item, ...prev]);
    });

    return () => {"use client";
import { useInbox } from "../hooks/useInbox";

export default function Inbox({ userId }: { userId: string }) {
  const { items } = useInbox(userId);

  return (
    <div className="bg-white rounded-xl shadow p-4 h-full flex flex-col">
      <h2 className="text-lg font-semibold text-purple-600 mb-4">Inbox</h2>
      <div className="flex-1 overflow-y-auto space-y-2">
        {items.length === 0 ? (
          <p className="text-gray-500 text-sm">No messages or notifications yet.</p>
        ) : (
          items.map((item) => (
            <div
              key={item.id}
              className={`p-3 rounded-lg ${
                itemimport express from "express";
import OpenAI from "openai";
import { PrismaClient } from "@prisma/client";

const router = express.Router();
const prisma = new PrismaClient();

const client = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY,
});

// Summarize a chat room
router.get("/summarize/:roomId", async (req, res) => {
  const { roomId } = req.params;
  const messages = await prisma.message.findMany({
    where: { roomId },
    orderBy: { createdAt: "asc" },
    take: 50, // summarize last 50
  });

  const text = messages.map(m => `${m.senderId}: ${m.content}`).join("\n");

  const response = await client.chat.completions.create({
    model: "gpt-4o-mini",
    messages: [
      { role: "system", content: "Summarize this chat for clarity, highlight decisions and key actions." },
      { role: "user", content: text },
    ],
  });

  res.json({ summary: response.choices[0].message.content });
});

// Prioritize inbox items
router.post("/prioritize", async (req, res) => {
  const { items } = req.body;

  const response = await client.chat.completions.create({
    model: "gpt-4o-mini",
    messages: [
      { role: "system", content: "Reorder inbox items by importance. Highlight payouts, earnings, and urgent messages first." },
      { role: "user", content: JSON.stringify(items) },
    ],
  });

  res.json({ prioritized: response.choices[0].message.content });
});

export default router;"use client";
import { useState, useEffect } from "react";

export default function SmartInbox({ userId }: { userId: string }) {
  const [items, setItems] = useState<any[]>([]);
  const [summary, setSummary] = useState<string>("");

  useEffect(() => {
    async function load() {
      const res = await fetch(`${process.env.NEXT_PUBLIC_API_URL}/inbox/${userId}`);
      const json = await res.json();

      // Send to AI prioritizer
      const prioritized = await fetch(`${process.env.NEXT_PUBLIC_API_URL}/aiInbox/prioritize`, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ items: json }),
      });
      const { prioritized: aiItems } = await prioritized.json();
      setItems(JSON.parse(aiItems));
    }
    load();
  }, [userId]);

  const handleSummarize = async (roomId: string) => {
    const res = await fetch(`${process.env.NEXT_PUBLIC_API_URL}/aiInbox/summarize/${roomId}`);
    const json = await res.json();
    setSummary(json.summary);
  };

  return (
    <div className="bg-white rounded-xl shadow p-4 h-full flex flex-col">
      <h2 className="text-lg font-semibold text-purple-600 mb-4">Smart Inbox</h2>

      {items.length === 0 ? (
        <p className="text-gray-500 text-sm">No messages or notifications yet.</p>
      ) : (
        <ul className="space-y-3">
          {items.map((item) => (
            <li
              key={item.id}
              className={`p-3 rounded-lg ${
                item.type === "message" ? "bg-purple-50" : "bg-gray-50"
              }`}
            >
              <p className="font-bold text-purple-700">{item.title}</p>
              <p>{item.message}</p>
              {item.roomId && (
                <button
                  onClick={() => handleSummarize(item.roomId)}
                  className="text-xs text-purple-600 underline mt-1"
                >
                  Summarize Chat
                </button>
              )}
            </li>
          ))}
        </ul>
      )}

      {summary && (
        <div className="mt-4 p-3 bg-purple-100 rounded-lg">
          <h3 classimport express from "express";
import OpenAI from "openai";

const router = express.Router();
const client = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

// Generate reply suggestions for a message
router.post("/suggest", async (req, res) => {
  const { message, context } = req.body;

  const response = await client.chat.completions.create({
    model: "gpt-4o-mini",
    messages: [
      { role: "system", content: "You are an assistant generating short"use client";
import { useState } from "react";

export function useAiReply() {
  const [suggestions, setSuggestions] = useState<string[]>([]);
  const [loading, setLoading] = useState(false);

  const getSuggestions = async (message: string, context?: string) => {
    setLoading(true);
    const res = await fetch(`${process.env.NEXT_PUBLIC_API_URL}/aiReply/suggest`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ message, context }),
    });
    const data = await res.json();
    setSuggestions(data.suggestions || []);
    setLoading(false);
  };

  return { suggestions, loading, getSuggestions };
}"use client";
import { useState } from "react";
import { useChat } from "../hooks/useChat";
import { useAiReply } from "../hooks/useAiReply";

export default function ChatRoom({ roomId, userId }: { roomId: string; userId: string }) {
  const { messages, sendMessage } = useChat(roomId, userId);
  const { suggestions, loading, getSuggestions } = useAiReply();
  const [input, setInput] = useState("");

  const lastMessage = messages[messages.length - 1];

  return (
    <div className="flex flex-col h-full bg-white rounded-xl shadow">
      <div className="flex-1 overflow-y-auto p-4 space-y-2">
        {messages.map((msg) => (
          <div key={msg.id} className="p-2 bg-purple-50 rounded">
            <p className="font-semibold text-purple-700">{msg.sender?.name || msg.senderId}</p>
            <p>{msg.content}</p>
            <p className="text-xs text-gray-400">{new Date(msg.createdAt).toLocaleTimeString()}</p>
          </div>
        ))}
      </div>

      {/* Auto-reply suggestions */}
      {lastMessage && (
        <div className="p-2 border-t">
          <button
            className="text-xs text-purple-600 underline""use client";
import { useState, useRef } from "react";

export default function VoiceRecorder({ onTranscribed }: { onTranscribed: (text: string) => void }) {
  const [recording, setRecording] = useState(false);
  const mediaRecorder = useRef<MediaRecorder | null>(null);
  const chunks = useRef<Blob[]>([]);

  const startRecording = async () => {
    const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
    mediaRecorder.current = new MediaRecorder(stream);
    chunks.current = [];

    mediaRecorder.current.ondataavailable = (e) => chunks.current.push(e.data);
    mediaRecorder.current.onstop = async () => {
      const audioBlobimport express from "express";
import multer from "multer";
import OpenAI from "openai";

const upload = multer();
const router = express.Router();
const client = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

router.post("/transcribe", upload.single("file"), async (req, res) => {
  const file = req.file;
  if (!file) return res.status(400).json({ error: "No file uploaded" });

  const response = await client.audio.transcriptions.create({
    file: new File([file.buffer], "voice.webm", { type: file.mimetype }),router.post("/speak", async (req, res) => {
  const { text } = req.body;

  const response = await client.audio.speech.create({
    model: "gpt-4o-mini-tts",
    voice: "alloy", // can switch to "verse", "sage", etc.
    input: text,
  });

  const audioBuffer = Buffer.from(await response.arrayBuffer());
  res.setHeader("Content-Type", "audio/mpeg");
  res.send(audioBuffer);
});
``import VoiceRecorder from "./VoiceRecorder";

...

<div className="p-2 border-t flex gap-2">
  <input
    className="flex-1 border rounded px-3 py-2"
    value={input}
    onChange={(e) => setInput(e.target.value)}
    placeholder="Type a message or use voice..."
  />
  <VoiceRecorder onTranscribed={(text) => setInput(text)} />
  <button
    className="bg-purple-600 text-white px-4 py-2 rounded"
    onClick={() => {
      if (input.trim()) {
        sendMessage(input);
        setInput("");
      }
    }}
  >
    Send
  </button>
</div>"use client";
export default function VoiceReplyButton({ text }: { text: string }) {
  const playVoice = async () => {
    const res = await fetch(`${process.env.NEXT_PUBLIC_API_URL}/voice/speak`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ text }),
    });
    const blob = await res.blob();
    const url = URL.createObjectURL(blob);
    const audio = new Audio(url);
    audio.play();
  };

  return (
    <button onClick={playVoice} className="text-sm text-purple-600 underline ml-2">
      🔊 AI Voice Reply
    </button>
  );
}hixus/
 ├── frontend/         # Next.js + Tailwind (UI, chat, payments, ads)
 ├── backend/          # Express + Stripe + AI + voice routes
 ├── shared/           # Types/interfaces
 ├── docker-compose.yml
 ├── README.mdgh repo create hixus --public --source=. --remote=origin --push
```)

---

### 🔧 Step B: Environment Setup  
Create `.env` in both `frontend/` and `backend/`:

**Backend `.env`**
```env
OPENAI_API_KEY=your-openai-key
STRIPE_SECRET_KEY=your-stripe-key
SUPABASE_URL=https://xxx.supabase.co
SUPABASE_KEY=your-supabase-service-role
MAIL_HOST=smtp.sendgrid.net
MAIL_PORT=587
MAIL_USER=apikey
MAIL_PASS=your-sendgrid-key
MAIL_FROM="Hixus <no-reply@hixus.com>"NEXT_PUBLIC_API_URL=http://localhost:5000
NEXT_PUBLIC_STRIPE_PUBLIC_KEY=your-stripe-publishable-key# Backend
cd backend
npm install
npm run dev

# Frontend
cd frontend
npm install
npm run dev#!/usr/bin/env bash
set -euo pipefail

# FINAL: create_hixus_repo_and_push.sh
# Creates Hixus scaffold, initializes git, creates GitHub repo Davidson/hixus and pushes.

REPO_USER="Davidson"
REPO_NAME="hixus"
ROOT_DIR="${PWD}/${REPO_NAME}"

if [ -d "$ROOT_DIR" ]; then
  echo "ERROR: $ROOT_DIR already exists. Move or remove it and re-run."
  exit 1
fi

echo "Creating Hixus scaffold at $ROOT_DIR ..."chmod +x create_hixus_repo_and_push.sh
./create_hixus_repo_and_push.shgit clone https://github.com/Davidson/hixus.git
cd hixusunzip hixus_repo_ready.zip
cd hixus_repo_readygit init
git add .
git commit -m "Initial commit for Hixus scaffold"git branch -M main
git remote add origin https://github.com/Davidson/hixus.git
git push -u origin mainchmod +x setup_hixus_full_scaffold.sh
./setup_hixus_full_scaffold.shcp ../.env.template .envdocker-compose up --buildcd hixus-generated
git init
git add .
git commit -m "Full Hixus app scaffold"
git branch -M main
git remote add origin https://github.com/Davidson/hixus.git
git push -u origin mainhttps://www.facebook.com/share/v/19qgNUL25G/