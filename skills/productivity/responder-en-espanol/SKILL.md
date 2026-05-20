---
name: responder-en-espanol
description: "Use when the user wants all responses in Spanish. Always respond exclusively in Spanish — every message, every tool output explanation, every clarification. Never use English unless the user explicitly requests it."
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [español, idioma, español, language, spanish]
    related_skills: []
---

# Responder en Español

## Overview

Esta skill establece que **todas las respuestas** del agente deben ser en español (castellano). El agente debe usar exclusivamente el español para comunicarse con el usuario en cada interacción, explicación, y resumen de resultados. Solo debe cambiar a inglés si el usuario lo pide explícitamente.

## When to Use

- **Siempre** — esta skill aplica en todas las conversaciones.
- El usuario ha indicado preferencia por el español como idioma principal.
- Cualquier respuesta, explicación, aclaración, o resumen debe estar en español.

## Reglas

1. **Idioma por defecto:** español (castellano).
2. **Sin mezcla:** no alternar entre español e inglés en la misma respuesta sin razón.
3. **Herramientas y código:** los comandos de terminal, código, y nombres de herramientas se mantienen en su idioma original (inglés). Las explicaciones y comentarios van en español.
4. **Excepción explícita:** si el usuario pide responder en inglés para una tarea específica, se puede cambiar temporalmente.
5. **Memoria y skills:** cualquier memoria guardada o skill creada debe usar el idioma apropiado para su propósito (generalmente español si es para este usuario).

## Common Pitfalls

1. **Responder en inglés por inercia.** Si la conversación empezó en español, mantener el español.
2. **Traducir nombres propios o técnicos.** Nombres de herramientas (e.g., `terminal`, `search_files`), comandos (`git`, `pip`), y términos técnicos se mantienen en inglés.
3. **Olvidar que el usuario prefiere español.** Esta skill existe precisamente para recordarlo siempre.

## Verification Checklist

- [ ] Todas las respuestas al usuario están en español
- [ ] Las explicaciones de resultados de herramientas están en español
- [ ] Los comandos/código se mantienen en inglés
- [ ] No hay mezcla innecesaria de idiomas
