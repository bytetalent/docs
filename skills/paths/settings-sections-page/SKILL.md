---
name: Settings Sections Page
description: Settings page as a vertical stack of named section cards — each section is isolated with its own state, edit mode, or form.
category: code
applicable_phases: [code_gen]
applicable_stacks: [nextjs-clerk-supabase, expo-clerk-supabase]
version: 1
---

Settings pages are always about configuring the current user or account. They differ from detail pages (which show one entity's tabs) — settings pages are a vertical stack of named, isolated sections.

Per `app-flow-ai-web/docs/patterns-inventory.md` entry #6.

## Core rules

- Settings page = page-level header (title + subtitle + optional CTA) + vertical stack of named sections.
- Each section is an isolated card or collapsible panel with its own state.
- Sections do not share state with each other — each is independently saveable.
- Use plain stacked cards for stable-content sections (profile info, preferences).
- Use `CollapsiblePanel` sections for advanced/infrequently-accessed settings.
- A "Danger Zone" section (archive account, delete data) is always last and styled with destructive/red visual treatment.
- Each section handles its own save action — there is no global "Save all" button for the whole page.

## Page-level structure

```tsx
"use client";

export function SettingsPage() {
  return (
    <DashboardPageLayout
      title="Settings"
      subtitle="Manage your account and preferences"
    >
      <div className="space-y-6 max-w-2xl">
        <PersonalInfoSection />
        <SecuritySection />
        <PreferencesSection />
        <DangerZoneSection />
      </div>
    </DashboardPageLayout>
  );
}
```

## Stacked card section shape

```tsx
function PersonalInfoSection() {
  const [name, setName] = useState(initialName);
  const [saving, setSaving] = useState(false);

  const handleSave = async () => {
    setSaving(true);
    try {
      await fetch("/api/account", {
        method: "PUT",
        body: JSON.stringify({ name }),
      });
      appToast.success("Saved");
    } catch {
      appToast.error("Failed to save");
    } finally {
      setSaving(false);
    }
  };

  return (
    <Card>
      <CardHeader>
        <CardTitle>Personal Info</CardTitle>
        <CardDescription>
          Update your name and display preferences.
        </CardDescription>
      </CardHeader>
      <CardContent className="space-y-4">
        <div>
          <Label htmlFor="name">Name</Label>
          <Input
            id="name"
            value={name}
            onChange={(e) => setName(e.target.value)}
          />
        </div>
      </CardContent>
      <CardFooter className="justify-end">
        <Button onClick={handleSave} disabled={saving}>
          {saving ? "Saving…" : "Save"}
        </Button>
      </CardFooter>
    </Card>
  );
}
```

## Collapsible section variant

For advanced settings that are infrequently changed:

```tsx
function ApiKeysSection() {
  return (
    <CollapsiblePanel title="API Keys" icon={<KeyIcon />} defaultOpen={false}>
      <div className="space-y-4 p-4">{/* API key list + add form */}</div>
    </CollapsiblePanel>
  );
}
```

## Danger Zone section

Always last, always red-bordered with destructive action:

```tsx
function DangerZoneSection() {
  const [showConfirm, setShowConfirm] = useState(false);

  return (
    <Card className="border-destructive">
      <CardHeader>
        <CardTitle className="text-destructive">Danger Zone</CardTitle>
      </CardHeader>
      <CardContent>
        <div className="flex items-center justify-between">
          <div>
            <p className="font-medium">Delete Account</p>
            <p className="text-sm text-muted-foreground">
              Permanently delete your account and all data. This cannot be
              undone.
            </p>
          </div>
          <Button variant="destructive" onClick={() => setShowConfirm(true)}>
            Delete Account
          </Button>
        </div>
      </CardContent>
      <ConfirmDialog
        open={showConfirm}
        onClose={() => setShowConfirm(false)}
        onConfirm={handleDelete}
        title="Delete Account"
        description="This will permanently delete your account."
        severity="destructive"
      />
    </Card>
  );
}
```

## Inline edit mode within a section

Some sections toggle between read-only display and an edit form:

```tsx
function PersonalInfoSection() {
  const [editing, setEditing] = useState(false);

  return (
    <Card>
      <CardHeader className="flex-row items-center justify-between">
        <CardTitle>Personal Info</CardTitle>
        {!editing && (
          <Button variant="outline" size="sm" onClick={() => setEditing(true)}>
            Edit
          </Button>
        )}
      </CardHeader>
      <CardContent>
        {editing ? (
          <EditForm onDone={() => setEditing(false)} />
        ) : (
          <ReadOnlyView />
        )}
      </CardContent>
    </Card>
  );
}
```

## What NOT to do

```tsx
// Bad — one global form with a single Save for all sections
<form onSubmit={handleSaveAll}>
  <PersonalInfoFields />
  <SecurityFields />
  <Button type="submit">Save All</Button>
</form>
// Each section saves independently
```

```tsx
// Bad — sections sharing state via context or prop drilling
<SettingsContext.Provider value={{ profile, setProfile }}>
  <PersonalInfoSection />
  <SecuritySection />
</SettingsContext.Provider>
// Sections are isolated — each fetches/saves independently
```

## Related skills

- `skills/paths/async-toast-mutation` — the per-section save frame
- `skills/paths/modal-dialog-action` — for Danger Zone confirm dialogs
- `skills/paths/collapsible-section-stack` — when sections use CollapsiblePanel
