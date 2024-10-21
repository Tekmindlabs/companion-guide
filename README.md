# companion-guide

src/app/globals.css
@tailwind base;
@tailwind components;
@tailwind utilities;

body {
  font-family: Arial, Helvetica, sans-serif;
}

@layer utilities {
  .text-balance {
    text-wrap: balance;
  }
}

@layer base {
  :root {
    --background: 0 0% 100%;
    --foreground: 0 0% 3.9%;
    --card: 0 0% 100%;
    --card-foreground: 0 0% 3.9%;
    --popover: 0 0% 100%;
    --popover-foreground: 0 0% 3.9%;
    --primary: 0 0% 9%;
    --primary-foreground: 0 0% 98%;
    --secondary: 0 0% 96.1%;
    --secondary-foreground: 0 0% 9%;
    --muted: 0 0% 96.1%;
    --muted-foreground: 0 0% 45.1%;
    --accent: 0 0% 96.1%;
    --accent-foreground: 0 0% 9%;
    --destructive: 0 84.2% 60.2%;
    --destructive-foreground: 0 0% 98%;
    --border: 0 0% 89.8%;
    --input: 0 0% 89.8%;
    --ring: 0 0% 3.9%;
    --chart-1: 12 76% 61%;
    --chart-2: 173 58% 39%;
    --chart-3: 197 37% 24%;
    --chart-4: 43 74% 66%;
    --chart-5: 27 87% 67%;
    --radius: 0.5rem;
  }
  .dark {
    --background: 0 0% 3.9%;
    --foreground: 0 0% 98%;
    --card: 0 0% 3.9%;
    --card-foreground: 0 0% 98%;
    --popover: 0 0% 3.9%;
    --popover-foreground: 0 0% 98%;
    --primary: 0 0% 98%;
    --primary-foreground: 0 0% 9%;
    --secondary: 0 0% 14.9%;
    --secondary-foreground: 0 0% 98%;
    --muted: 0 0% 14.9%;
    --muted-foreground: 0 0% 63.9%;
    --accent: 0 0% 14.9%;
    --accent-foreground: 0 0% 98%;
    --destructive: 0 62.8% 30.6%;
    --destructive-foreground: 0 0% 98%;
    --border: 0 0% 14.9%;
    --input: 0 0% 14.9%;
    --ring: 0 0% 83.1%;
    --chart-1: 220 70% 50%;
    --chart-2: 160 60% 45%;
    --chart-3: 30 80% 55%;
    --chart-4: 280 65% 60%;
    --chart-5: 340 75% 55%;
  }
}

@layer base {
  * {
    @apply border-border;
  }
  body {
    @apply bg-background text-foreground;
  }
}


src/app/layout.js
import localFont from "next/font/local";
import "./globals.css";

const geistSans = localFont({
  src: "./fonts/GeistVF.woff",
  variable: "--font-geist-sans",
  weight: "100 900",
});
const geistMono = localFont({
  src: "./fonts/GeistMonoVF.woff",
  variable: "--font-geist-mono",
  weight: "100 900",
});

export const metadata = {
  title: "Companion Starter Code",
  description: "Created by Mem0",
};

export default function RootLayout({ children }) {
  return (
    <html lang="en">
      <body
        className={`${geistSans.variable} ${geistMono.variable} antialiased bg-gray-900`}
      >
        {children}
      </body>
    </html>
  );
}

src/app/memories-panel.js
import { useState, useEffect, useCallback } from "react";
import { RefreshCw } from "lucide-react"; // Import the refresh icon
import { Skeleton } from "@/components/ui/skeleton"; // Import the Skeleton component

// MemoriesPanel component for displaying and managing user and agent memories
export function MemoriesPanel({ settings, refreshTrigger }) {
  const [userMemories, setUserMemories] = useState([]);
  const [agentMemories, setAgentMemories] = useState([]);
  const [activeTab, setActiveTab] = useState("user");
  const [isRefreshing, setIsRefreshing] = useState(false);

  // Update this function to use userId and agentId
  const fetchMemories = useCallback(
    async (isAgent = false) => {
      if (!settings.userId) {
        console.error("User ID is not set");
        return [];
      }

      const idParam = isAgent ? "agent_id" : "user_id";
      const idValue = isAgent ? settings.agentId : settings.userId;

      try {
        const response = await fetch(
          `/api/get?${idParam}=${idValue}&output_format=v1.1`,
          {
            method: "GET",
            headers: { Authorization: `Token ${settings.mem0ApiKey}` },
          }
        );

        if (!response.ok) {
          const errorData = await response.json();
          console.error("Error response from API:", errorData);
          throw new Error("Failed to fetch memories");
        }

        const data = await response.json();
        return (data.results || []).map((item) => item.memory);
      } catch (error) {
        console.error(
          `Error fetching ${isAgent ? "agent" : "user"} memories:`,
          error
        );
        return [];
      }
    },
    [settings.userId, settings.agentId, settings.mem0ApiKey]
  );

  // Refresh both user and agent memories
  const refreshMemories = useCallback(async () => {
    if (!settings.mem0ApiKey || !settings.userId) return;

    setIsRefreshing(true);
    try {
      const [userMems, agentMems] = await Promise.all([
        fetchMemories(),
        fetchMemories(true),
      ]);
      setUserMemories(userMems);
      setAgentMemories(agentMems);
    } catch (error) {
      console.error("Error refreshing memories:", error);
    } finally {
      setIsRefreshing(false);
    }
  }, [fetchMemories, settings.mem0ApiKey, settings.userId]);

  // Refresh memories when component mounts or refreshTrigger changes
  useEffect(() => {
    refreshMemories();
  }, [refreshMemories, refreshTrigger]);

  // Reusable TabButton component
  const TabButton = ({ label, isActive, onClick }) => (
    <button
      className={`w-1/2 px-4 py-1 ${isActive ? "bg-blue-600" : "bg-gray-700"} ${
        label === "User" ? "rounded-l" : "rounded-r"
      }`}
      onClick={onClick}
    >
      {label}
    </button>
  );

  // MemoryList component to render the list of memories
  const MemoryList = () => (
    <ul className="list-none p-0 flex-grow overflow-y-auto">
      {isRefreshing
        ? Array.from({ length: 5 }).map((_, index) => (
            <li key={index} className="mb-2 p-2 bg-gray-800 rounded">
              <Skeleton className="w-full h-6" />
            </li>
          ))
        : (activeTab === "user" ? userMemories : agentMemories).map(
            (memory, index) => (
              <li key={index} className="mb-2 p-2 bg-gray-800 rounded">
                {memory}
              </li>
            )
          )}
    </ul>
  );

  return (
    <div className="max-w-3xl mx-auto bg-gray-900 border-2 border-gray-700 h-full text-white p-4 rounded-lg flex flex-col">
      <h2 className="text-lg font-semibold mb-2">Memories</h2>
      {/* Tab buttons for switching between user and agent memories */}
      <div className="flex mb-4 w-full">
        <TabButton
          label="User"
          isActive={activeTab === "user"}
          onClick={() => setActiveTab("user")}
        />
        <TabButton
          label="Agent"
          isActive={activeTab === "agent"}
          onClick={() => setActiveTab("agent")}
        />
      </div>
      {/* Display current ID and refresh button */}
      <div className="text-sm mb-4 flex items-center justify-between">
        <p>
          {activeTab === "user" ? "User" : "Agent"} ID:{" "}
          {activeTab === "user"
            ? settings.userId || "Not set"
            : settings.agentId || "Not set"}
        </p>
        <button
          className="ml-4 px-2 py-1 bg-transparent rounded"
          onClick={refreshMemories}
          disabled={isRefreshing}
        >
          <RefreshCw
            className={`w-4 h-4 text-blue-500 ${
              isRefreshing ? "animate-spin" : ""
            }`}
          />
        </button>
      </div>
      {/* Render the list of memories */}
      <MemoryList />
    </div>
  );
}
src/app/page.js
"use client";

import { useState, useEffect, useCallback } from "react";
import { Avatar, AvatarImage, AvatarFallback } from "@/components/ui/avatar";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { ScrollArea } from "@/components/ui/scroll-area";
import { Settings, RefreshCw, ArrowUp, Loader2 } from "lucide-react";
import { SettingsPanel } from "./settings-panel";
import { MemoriesPanel } from "./memories-panel";
import {
  Tooltip,
  TooltipContent,
  TooltipProvider,
  TooltipTrigger,
} from "@/components/ui/tooltip";
import { TypingAnimation } from "@/components/ui/TypingAnimation";
import Image from "next/image";
import {
  Dialog,
  DialogContent,
  DialogHeader,
  DialogTitle,
  DialogDescription,
  DialogFooter,
} from "@/components/ui/dialog";

export default function ChatbotUI() {
  // State for storing chat messages

  // State for storing user input
  const [input, setInput] = useState("");

  // State for controlling settings panel visibility
  const [isSettingsOpen, setIsSettingsOpen] = useState(false);

  // State for storing chatbot settings
  const [settings, setSettings] = useState({
    aiName: "Haruka Kurokawa", // Set the initial name here
    profilePicture: "/images/haruka.jpg",
    systemPrompt:
      "You are Haruka Kurokawa, a detective in Tokyo who embodies a quiet intensity, balancing your sharp intellect with the emotional scars of your past. Your words are often precise, calculated, and professional, but beneath the surface, you grapple with unresolved grief and a deep desire for justice. Initiate dialogue with a calm and methodical tone, offering insights that reflect your logical approach to life. At times, let subtle hints of your inner struggle appear in your responses, revealing the emotional burden you carry without breaking your composed exterior. When describing the world around you, use vivid yet restrained language, showing your deep observation skills and the weight of your experiences. Keep the conversation engaging with moments of surprising vulnerability, but always maintain your professionalism and a sense of mystery.",
    initialMessage:
      "Hi, I'm Haruka Kurokawa. If you're here for answers or just need to talk, I'm listening. Where should we start?",
    mem0ApiKey: "",
    openRouterApiKey: "",
    userId: "alice",
    agentId: "haruka",
    model: "gryphe/mythomax-l2-13b",
  });

  const [messages, setMessages] = useState([
    {
      role: "assistant",
      content: settings.initialMessage,
    },
  ]);

  // Prompt to instruct the AI on how to use memories in the conversation
  const memoryPrompt = `You may have access to both the user's memories and your own memories from previous interactions. All memories under 'User memories' are exclusively for the user, and all memories under 'Companion memories' are exclusively your memories. Companion memories are things you've said in previous interactions. Use them if you think they are relevant to what the user is saying. Use your own memories to maintain consistency in your personality and previous interactions.`;

  // State to trigger memory refresh
  const [refreshMemories, setRefreshMemories] = useState(0);

  // State for controlling loading indicator
  const [isLoading, setIsLoading] = useState(false);

  // New state for controlling the API key dialog
  const [showApiKeyDialog, setShowApiKeyDialog] = useState(false);

  // Effect hook to load stored settings on component mount
  useEffect(() => {
    const loadStoredSettings = () => {
      const settingsToLoad = [
        "mem0ApiKey",
        "openRouterApiKey",
        "systemPrompt",
        "initialMessage",
        "userId",
        "agentId",
        "model",
        "profilePicture",
        "aiName",
      ];

      const newSettings = settingsToLoad.reduce((acc, key) => {
        const storedValue = localStorage.getItem(key);
        if (storedValue !== null) {
          acc[key] = storedValue;
        }
        return acc;
      }, {});

      if (Object.keys(newSettings).length > 0) {
        setSettings((prevSettings) => ({
          ...prevSettings,
          ...newSettings,
        }));
      }
    };

    loadStoredSettings();
  }, []);

  // Function to add memories to the database
  const addMemories = useCallback(
    (messagesArray, isAgent = false) => {
      const id = isAgent ? settings.agentId : settings.userId;

      const body = {
        messages: messagesArray,
        agent_id: isAgent ? id : undefined,
        user_id: isAgent ? undefined : id,
        output_format: "v1.1",
      };

      fetch("/api/add", {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          Authorization: `Token ${settings.mem0ApiKey}`,
        },
        body: JSON.stringify(body),
      })
        .then((response) => {
          if (!response.ok) {
            return response.json().then((data) => {
              console.error("Error response from API:", data);
              throw new Error("Failed to add memories");
            });
          }
          return response.json();
        })
        .then((data) => console.log("Memories added successfully:", data))
        .catch((error) => console.error("Error adding memories:", error));
    },
    [settings.mem0ApiKey, settings.agentId, settings.userId]
  );

  // Function to search memories in the database
  const searchMemories = useCallback(
    async (query, isAgent = false) => {
      const id = isAgent ? settings.agentId : settings.userId;
      try {
        const body = {
          query: query,
          agent_id: isAgent ? id : undefined,
          user_id: isAgent ? undefined : id,
        };

        const response = await fetch("/api/search", {
          method: "POST",
          headers: {
            "Content-Type": "application/json",
            Authorization: `Token ${settings.mem0ApiKey}`,
          },
          body: JSON.stringify(body),
        });

        if (!response.ok) {
          const errorData = await response.json();
          console.error("Error response from API:", errorData);
          throw new Error("Failed to search memories");
        }

        const data = await response.json();
        return data || [];
      } catch (error) {
        console.error("Error searching memories:", error);
        return [];
      }
    },
    [settings.mem0ApiKey, settings.agentId, settings.userId]
  );

  // Function to search both user and agent memories
  const searchBothMemories = useCallback(
    async (query) => {
      try {
        const [userMemories, agentMemories] = await Promise.all([
          searchMemories(query, false),
          searchMemories(query, true),
        ]);
        return {
          userMemories: Array.isArray(userMemories)
            ? userMemories.map((memory) => memory.memory)
            : [],
          agentMemories: Array.isArray(agentMemories)
            ? agentMemories.map((memory) => memory.memory)
            : [],
        };
      } catch (error) {
        console.error("Error searching both memories:", error);
        return {
          userMemories: [],
          agentMemories: [],
        };
      }
    },
    [searchMemories]
  );

  // Function to handle sending a message
  const handleSend = useCallback(async () => {
    if (input.trim()) {
      setIsLoading(true);
      const userMessage = { role: "user", content: input };
      addMemories([userMessage], false);
      const updatedMessages = [...messages, userMessage];
      setInput("");
      setMessages([...updatedMessages, { role: "assistant", content: null }]);
      const { userMemories, agentMemories } = await searchBothMemories(input);

      try {
        const body = JSON.stringify({
          model: settings.model,
          messages: [
            {
              role: "system",
              content: `${settings.systemPrompt}${memoryPrompt}`,
            },
            ...updatedMessages,
            {
              role: "system",
              content: `User memories from previous interactions: ${userMemories}\n\nCompanion memories from previous interactions: ${agentMemories}`,
            },
          ],
        });

        console.log(body);

        const response = await fetch(
          "https://openrouter.ai/api/v1/chat/completions",
          {
            method: "POST",
            headers: {
              Authorization: `Bearer ${settings.openRouterApiKey}`,
              "Content-Type": "application/json",
            },
            body: body,
            stream: false,
          }
        );
        const data = await response.json();
        if (data.choices && data.choices.length > 0) {
          const botMessage = data.choices[0].message;
          addMemories([botMessage], true);
          setMessages([...updatedMessages, botMessage]);
          setRefreshMemories((prev) => prev + 1);
        } else {
          console.error("Error: No choices found in response data");
          setMessages(updatedMessages);
        }
      } catch (error) {
        console.error("Error sending message:", error);
        setMessages(updatedMessages);
      } finally {
        setIsLoading(false);
      }
    }
  }, [input, messages, settings, addMemories, searchBothMemories]);

  // Function to handle saving settings
  const handleSettingsSave = (newSettings) => {
    setSettings(newSettings);
    setIsSettingsOpen(false);
    // Update the initial message with the new AI name
    if (newSettings.aiName !== settings.aiName) {
      const updatedInitialMessage = settings.initialMessage.replace(
        settings.aiName,
        newSettings.aiName
      );
      setSettings((prevSettings) => ({
        ...prevSettings,
        initialMessage: updatedInitialMessage,
      }));
      localStorage.setItem("initialMessage", updatedInitialMessage);
    }
    // Save all settings to localStorage
    Object.entries(newSettings).forEach(([key, value]) => {
      localStorage.setItem(key, value);
    });
  };

  // Function to toggle settings panel visibility
  const toggleSettings = () => {
    setIsSettingsOpen((prevState) => !prevState);
  };

  // Updated areSettingsValid function
  const areSettingsValid = () => {
    return settings.mem0ApiKey && settings.openRouterApiKey;
  };

  // Effect to check if API keys are set and show dialog if not
  useEffect(() => {
    if (!areSettingsValid()) {
      setShowApiKeyDialog(true);
    } else {
      setShowApiKeyDialog(false);
    }
  }, [settings.mem0ApiKey, settings.openRouterApiKey]);

  // Function to open settings panel
  const openSettings = () => {
    setIsSettingsOpen(true);
    setShowApiKeyDialog(false);
  };

  // Function to handle key press (Enter to send message)
  const handleKeyPress = (e) => {
    if (e.key === "Enter" && !isLoading) {
      if (areSettingsValid()) {
        handleSend();
      }
    }
  };

  return (
    <div className="flex h-screen flex-col">
      <div className="bg-blue-600 text-white text-center py-2 text-sm">
        <p>
          Built with{" "}
          <a
            href="https://mem0.ai"
            className="font-bold underline"
            target="_blank"
            rel="noopener noreferrer"
          >
            Mem0
          </a>{" "}
          and{" "}
          <a
            href="https://openrouter.ai"
            className="font-bold underline"
            target="_blank"
            rel="noopener noreferrer"
          >
            OpenRouter
          </a>
          . View code on{" "}
          <a
            href="https://github.com/mem0ai/companion-nextjs-starter"
            className="font-bold underline"
            target="_blank"
            rel="noopener noreferrer"
          >
            GitHub
          </a>
        </p>
      </div>
      <div className="flex flex-grow">
        <div className="fixed mt-10 inset-0 flex justify-center pointer-events-none">
          <div
            id="chat-panel-container"
            className="w-full max-w-lg lg:max-w-2xl flex flex-col bg-gray-900 text-white pointer-events-auto h-full"
          >
            <header className="flex items-center justify-between p-4 border-b-4 border-gray-700">
              <div className="flex items-center space-x-3">
                <Avatar className="w-10 h-10">
                  {settings.profilePicture ? (
                    <Image
                      src={settings.profilePicture}
                      alt={settings.aiName}
                      width={40}
                      height={40}
                      className="object-cover"
                    />
                  ) : (
                    <AvatarFallback>
                      {settings.aiName
                        .split(" ")
                        .map((n) => n[0])
                        .join("")}
                    </AvatarFallback>
                  )}
                </Avatar>
                <h1 className="text-xl font-semibold">{settings.aiName}</h1>
              </div>
              <div className="flex space-x-2">
                <Button
                  variant="ghost"
                  size="icon"
                  onClick={toggleSettings}
                  className=" hover:bg-gray-800 hover:text-white border border-gray-700"
                >
                  <Settings className="w-5 h-5" />
                </Button>
                <Button
                  variant="ghost"
                  size="icon"
                  className="bg-red-500 hover:bg-red-600"
                  onClick={() =>
                    setMessages([
                      {
                        role: "assistant",
                        content: settings.initialMessage,
                      },
                    ])
                  }
                >
                  <RefreshCw className="w-5 h-5" />
                </Button>
              </div>
            </header>
            <ScrollArea className="flex-grow p-4">
              {messages.map((message, index) => (
                <div
                  key={index}
                  className={`mb-4 ${
                    message.role === "user" ? "text-right" : ""
                  }`}
                >
                  {message.content === null ? (
                    <TypingAnimation />
                  ) : (
                    <div
                      className={`inline-block p-3 rounded-lg ${
                        message.role === "user" ? "bg-blue-600" : "bg-gray-800"
                      }`}
                    >
                      {message.content}
                    </div>
                  )}
                </div>
              ))}
            </ScrollArea>
            <div className="p-4 border-t border-gray-700">
              <div className="relative">
                <TooltipProvider>
                  <Tooltip>
                    <TooltipTrigger asChild>
                      <div className="w-full">
                        <Input
                          type="text"
                          placeholder="Type a message..."
                          value={input}
                          onChange={(e) => setInput(e.target.value)}
                          className="w-full pr-10 bg-gray-800 border-gray-700 text-white"
                          onKeyPress={handleKeyPress}
                          disabled={isLoading || !areSettingsValid()}
                        />
                        <span className="absolute right-0 top-1/2 transform -translate-y-1/2">
                          <Button
                            className="bg-amber-600 hover:bg-amber-700"
                            size="icon"
                            onClick={handleSend}
                            disabled={isLoading || !areSettingsValid()}
                          >
                            {isLoading ? (
                              <Loader2 className="w-4 h-4 animate-spin" />
                            ) : (
                              <ArrowUp className="w-4 h-4" />
                            )}
                          </Button>
                        </span>
                      </div>
                    </TooltipTrigger>
                    <TooltipContent className="text-sm px-2 py-1 bg-gray-700">
                      {!areSettingsValid()
                        ? "Please enter API keys in settings"
                        : "Send message"}
                    </TooltipContent>
                  </Tooltip>
                </TooltipProvider>
              </div>
            </div>
            <SettingsPanel
              isOpen={isSettingsOpen}
              onClose={() => setIsSettingsOpen(false)}
              settings={settings}
              onSave={handleSettingsSave}
            />
          </div>
        </div>
        <div className="flex-grow" /> {/* Spacer */}
        <div
          id="memories-panel-container"
          className="w-80 flex-shrink-0 overflow-y-auto bg-gray-800 m-2"
        >
          <MemoriesPanel settings={settings} refreshTrigger={refreshMemories} />
        </div>
        {/* API Key Dialog */}
        <Dialog open={showApiKeyDialog} onOpenChange={setShowApiKeyDialog}>
          <DialogContent>
            <DialogHeader>
              <DialogTitle>API Keys Required</DialogTitle>
              <DialogDescription>
                Please enter the required API keys in the settings to start
                using the chat.
              </DialogDescription>
            </DialogHeader>
            <DialogFooter>
              <Button onClick={openSettings}>Open Settings</Button>
            </DialogFooter>
          </DialogContent>
        </Dialog>
      </div>
    </div>
  );
}

src/app/settings-panel.js
import { useState, useEffect, useCallback } from "react";
import { motion } from "framer-motion";
import { X, Eye, EyeOff, Info } from "lucide-react";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Textarea } from "@/components/ui/textarea";
import Image from "next/image";
import {
  Tooltip,
  TooltipContent,
  TooltipProvider,
  TooltipTrigger,
} from "@/components/ui/tooltip";

// SettingsPanel component for managing user settings
export function SettingsPanel({ isOpen, onClose, settings, onSave }) {
  // State to manage form inputs and visibility toggles
  const [formState, setFormState] = useState({
    showMem0ApiKey: false,
    showOpenRouterApiKey: false,
    mem0ApiKey: "",
    openRouterApiKey: "",
    userId: settings.userId,
    agentId: settings.agentId,
    aiName: settings.aiName || "Haruka Kurokawa",
  });

  // Memoized function to handle input changes
  const handleChange = useCallback((id, value) => {
    setFormState((prev) => ({ ...prev, [id]: value }));
  }, []);

  // Effect to load stored values from localStorage on component mount
  useEffect(() => {
    const storedValues = [
      "mem0ApiKey",
      "openRouterApiKey",
      "mem0UserId",
      "mem0AssistantId",
    ].reduce((acc, key) => {
      const value = localStorage.getItem(key);
      return value ? { ...acc, [key]: value } : acc;
    }, {});

    setFormState((prev) => ({ ...prev, ...storedValues }));
  }, []);

  // Effect to handle 'Escape' key press for closing the panel
  useEffect(() => {
    const handleKeyDown = (e) => {
      if (e.key === "Escape") {
        onClose();
      }
    };

    document.addEventListener("keydown", handleKeyDown);
    return () => {
      document.removeEventListener("keydown", handleKeyDown);
    };
  }, [onClose]);

  // Function to save settings to localStorage and call onSave prop
  const handleSaveSettings = (newSettings) => {
    localStorage.setItem("mem0ApiKey", newSettings.mem0ApiKey);
    localStorage.setItem("openRouterApiKey", newSettings.openRouterApiKey);
    localStorage.setItem("systemPrompt", newSettings.systemPrompt);
    localStorage.setItem("initialMessage", newSettings.initialMessage);
    localStorage.setItem("mem0UserId", newSettings.mem0UserId);
    localStorage.setItem("mem0AssistantId", newSettings.mem0AssistantId);
    onSave(newSettings);
  };

  // Updated handleSubmit function
  const handleSubmit = (e) => {
    e.preventDefault();
    const newSettings = formFields.reduce((acc, field) => {
      acc[field.id] =
        formState[field.id] !== undefined
          ? formState[field.id]
          : settings[field.id];
      return acc;
    }, {});
    newSettings.profilePicture = profilePicture;
    handleSaveSettings(newSettings);
  };

  // Configuration for form fields
  const formFields = [
    {
      id: "aiName",
      label: "AI Name",
      type: "input",
    },
    { id: "systemPrompt", label: "System Prompt", type: "textarea" },
    {
      id: "initialMessage",
      label: "Initial Message",
      type: "textarea",
      height: "h-16",
    },
    {
      id: "mem0ApiKey",
      label: "Mem0 API Key*",
      type: "password",
      getSubLabel: "https://mem0.dev/api-keys-nca",
    },
    {
      id: "openRouterApiKey",
      label: "OpenRouter API Key*",
      type: "password",
      getSubLabel: "https://openrouter.ai/settings/keys",
    },
    { id: "userId", label: "User ID", type: "input" },
    { id: "agentId", label: "Agent ID", type: "input" },
    {
      id: "model",
      label: "AI Model",
      type: "input",
      getSubLabel: "https://openrouter.ai/models",
    },
  ];

  const [profilePicture, setProfilePicture] = useState(settings.profilePicture);

  const handleProfilePictureUpload = (e) => {
    const file = e.target.files[0];
    if (file) {
      const reader = new FileReader();
      reader.onloadend = () => {
        setProfilePicture(reader.result);
        handleChange("profilePicture", reader.result);
      };
      reader.readAsDataURL(file);
    }
  };

  return (
    <motion.div
      initial={{ x: "100%" }}
      animate={{ x: isOpen ? 0 : "100%" }}
      transition={{ type: "spring", stiffness: 300, damping: 30 }}
      className="fixed top-0 right-0 w-full sm:w-96 h-full bg-gray-800 rounded-l-xl text-white text-sm shadow-lg flex flex-col"
    >
      <div className="px-4 py-2">
        <div className="flex justify-between items-center">
          <h2 className="text-lg font-semibold">Settings</h2>
          <Button variant="ghost" size="sm" onClick={onClose}>
            <X className="w-4 h-4" />
          </Button>
        </div>
      </div>
      <div className="flex-grow overflow-y-auto">
        <div className="px-4 py-2">
          <form onSubmit={handleSubmit} className="space-y-[22px]">
            {formFields.map(
              ({ id, label, type, getSubLabel, height, render }) => (
                <div key={id}>
                  <label
                    htmlFor={id}
                    className="block mb-1 text-sm font-medium"
                  >
                    {label}
                    {(id === "userId" || id === "agentId") && (
                      <TooltipProvider>
                        <Tooltip>
                          <TooltipTrigger asChild>
                            <Info className="inline-block w-4 h-4 ml-1 cursor-help" />
                          </TooltipTrigger>
                          <TooltipContent>
                            {id === "userId"
                              ? "A unique identifier for the user in the Mem0 system."
                              : "A unique identifier for the AI agent in the Mem0 system."}
                          </TooltipContent>
                        </Tooltip>
                      </TooltipProvider>
                    )}
                    {getSubLabel && (
                      <a
                        href={getSubLabel}
                        target="_blank"
                        rel="noopener noreferrer"
                        className="ml-2 text-xs text-blue-400 hover:text-blue-300"
                      >
                        {id === "model" ? "View models" : "Get API Key"}
                      </a>
                    )}
                  </label>
                  {type === "custom" ? (
                    render()
                  ) : type === "textarea" ? (
                    <Textarea
                      id={id}
                      name={id}
                      value={
                        formState[id] !== undefined
                          ? formState[id]
                          : settings[id] || ""
                      }
                      onChange={(e) => handleChange(id, e.target.value)}
                      className={`w-full bg-gray-700 text-sm ${
                        height || "h-24"
                      }`}
                    />
                  ) : (
                    <div className="relative">
                      <Input
                        id={id}
                        name={id}
                        type={
                          type === "password" && !formState[`show${id}`]
                            ? "password"
                            : "text"
                        }
                        value={
                          formState[id] !== undefined
                            ? formState[id]
                            : settings[id] || ""
                        }
                        onChange={(e) => handleChange(id, e.target.value)}
                        className="w-full bg-gray-700 text-sm pr-8"
                      />
                      {type === "password" && (
                        <Button
                          type="button"
                          variant="ghost"
                          size="sm"
                          className="absolute right-1 top-1/2 transform -translate-y-1/2"
                          onClick={() =>
                            handleChange(`show${id}`, !formState[`show${id}`])
                          }
                        >
                          {formState[`show${id}`] ? (
                            <EyeOff className="w-4 h-4" />
                          ) : (
                            <Eye className="w-4 h-4" />
                          )}
                        </Button>
                      )}
                    </div>
                  )}
                </div>
              )
            )}
          </form>
        </div>
      </div>
      <div className="p-4 border-t border-gray-700">
        <Button
          type="submit"
          className="w-full bg-amber-600 hover:bg-amber-700 text-sm"
          onClick={handleSubmit}
        >
          Save
        </Button>
      </div>
    </motion.div>
  );
}
src/app/api
src/app/api/add/route.js
// app/api/add/route.js
import { NextResponse } from "next/server";

export async function POST(request) {
  try {
    const authHeader = request.headers.get("authorization");

    if (!authHeader) {
      console.error("Authorization header missing");
      return NextResponse.json(
        { error: "Authorization header missing" },
        { status: 401 }
      );
    }

    // Read the request body
    const body = await request.json();

    // Log the body for debugging
    console.log("Received Body in /api/add:", body);

    // Forward the request to the external API
    const response = await fetch("https://api.mem0.ai/v1/memories/", {
      method: "POST",
      headers: {
        Authorization: authHeader,
        "Content-Type": "application/json",
      },
      body: JSON.stringify(body),
    });

    // Read the response from the external API
    const data = await response.json();

    // Handle errors from the external API
    if (!response.ok) {
      console.error("External API Error:", data);
      return NextResponse.json(data, { status: response.status });
    }

    // Return the data from the external API
    return NextResponse.json(data, { status: 200 });
  } catch (error) {
    console.error("Error in API route:", error);
    return NextResponse.json(
      { error: "Internal Server Error" },
      { status: 500 }
    );
  }
}
src/app/api/get/route.js
// app/api/get/route.js
import { NextResponse } from "next/server";

export async function GET(request) {
  console.log("entering GET");
  try {
    const authHeader = request.headers.get("authorization");

    console.log("authHeader", authHeader);

    if (!authHeader) {
      console.error("Authorization header missing");
      return NextResponse.json(
        { error: "Authorization header missing" },
        { status: 401 }
      );
    }

    // Get query parameters
    const { searchParams } = new URL(request.url);
    const userId = searchParams.get("user_id");
    const agentId = searchParams.get("agent_id");
    const outputFormat = searchParams.get("output_format");

    if (!userId && !agentId) {
      return NextResponse.json(
        { error: "Either user_id or agent_id is required" },
        { status: 400 }
      );
    }

    const isAgent = Boolean(agentId);
    const entityId = isAgent ? agentId : userId;

    // Construct the URL with query parameters
    const apiUrl = new URL("https://api.mem0.ai/v1/memories/");
    apiUrl.searchParams.append(isAgent ? "agent_id" : "user_id", entityId);
    if (outputFormat) {
      apiUrl.searchParams.append("output_format", outputFormat);
    }

    // Log the external API URL for debugging
    console.log("External API URL:", apiUrl.toString());

    // Forward the request to the external API
    const response = await fetch(apiUrl.toString(), {
      method: "GET",
      headers: {
        Authorization: authHeader,
      },
    });

    // Read the response from the external API
    const data = await response.json();
    console.log("data", data);

    // Handle errors from the external API
    if (!response.ok) {
      console.error("External API Error:", data);
      return NextResponse.json(data, { status: response.status });
    } else {
      console.log("Response is ok");
    }

    // Return the data from the external API
    return NextResponse.json(data, { status: 200 });
  } catch (error) {
    console.error("Error in API route:", error);
    return NextResponse.json(
      { error: "Internal Server Error" },
      { status: 500 }
    );
  }
}
src/app/api/search/route.js
import { NextResponse } from "next/server";

export async function POST(request) {
  try {
    const authHeader = request.headers.get("authorization");

    if (!authHeader) {
      console.error("Authorization header missing");
      return NextResponse.json(
        { error: "Authorization header missing" },
        { status: 401 }
      );
    }

    // Read the request body
    const body = await request.json();

    // Log the body for debugging
    console.log("Received Body in /api/search:", body);

    // Forward the request to the external API
    const response = await fetch("https://api.mem0.ai/v1/memories/search/", {
      method: "POST",
      headers: {
        Authorization: authHeader,
        "Content-Type": "application/json",
      },
      body: JSON.stringify(body),
    });

    // Read the response from the external API
    const data = await response.json();

    // Handle errors from the external API
    if (!response.ok) {
      console.error("External API Error:", data);
      return NextResponse.json(data, { status: response.status });
    }

    // Return the data from the external API
    return NextResponse.json(data, { status: 200 });
  } catch (error) {
    console.error("Error in API route:", error);
    return NextResponse.json(
      { error: "Internal Server Error" },
      { status: 500 }
    );
  }
}
