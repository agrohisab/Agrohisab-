-- =====================================================
-- Agro Hisab: 0001 core schema (Phase 1)
-- =====================================================

-- ---------- profiles ----------
create table public.profiles (
  id uuid primary key references auth.users(id) on delete cascade,
  full_name text,
  phone text,
  preferred_language text not null default 'en'
    check (preferred_language in ('en','hi','gu','mr','bn')),
  created_at timestamptz not null default now()
);

-- ---------- shops (tenants) ----------
create table public.shops (
  id uuid primary key default gen_random_uuid(),
  name text not null check (char_length(trim(name)) between 2 and 120),
  owner_id uuid not null references auth.users(id) on delete restrict,
  phone text,
  address text,
  village_city text,
  district text,
  state text,
  pincode text check (pincode is null or pincode ~ '^[1-9][0-9]{5}$'),
  gstin text,
  pesticide_license_no text,
  fertilizer_license_no text,
  seed_license_no text,
  invoice_prefix text not null default 'INV'
    check (char_length(invoice_prefix) between 1 and 10
           and invoice_prefix ~ '^[A-Za-z0-9-]+$'),
  currency text not null default 'INR' check (currency = 'INR'),
  status text not null default 'active' check (status in ('active','suspended')),
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

-- ---------- shop_members ----------
create table public.shop_members (
  id uuid primary key default gen_random_uuid(),
  shop_id uuid not null references public.shops(id) on delete cascade,
  user_id uuid not null references auth.users(id) on delete cascade,
  role text not null check (role in ('owner','manager','staff')),
  is_active boolean not null default true,
  created_at timestamptz not null default now(),
  unique (shop_id, user_id)
);

create index shop_members_user_idx on public.shop_members (user_id) where is_active;
create index shops_owner_idx on public.shops (owner_id);

-- ---------- updated_at trigger ----------
create or replace function public.set_updated_at()
returns trigger
language plpgsql
set search_path = ''
as $$
begin
  new.updated_at = now();
  return new;
end;
$$;

create trigger shops_set_updated_at
before update on public.shops
for each row execute function public.set_updated_at();

-- ---------- auto-create profile on signup ----------
create or replace function public.handle_new_user()
returns trigger
language plpgsql
security definer
set search_path = ''
as $$
begin
  insert into public.profiles (id, full_name)
  values (new.id, nullif(trim(new.raw_user_meta_data ->> 'full_name'), ''))
  on conflict (id) do nothing;
  return new;
end;
$$;

create trigger on_auth_user_created
after insert on auth.users
for each row execute function public.handle_new_user();

-- ---------- turn RLS on immediately ----------
alter table public.profiles     enable row level security;
alter table public.shops        enable row level security;
alter table public.shop_members enable row level security;

-- ---------- table privileges (defense in depth on top of RLS) ----------
revoke all on public.profiles, public.shops, public.shop_members from anon;
revoke all on public.profiles, public.shops, public.shop_members from authenticated;

grant select on public.profiles, public.shops, public.shop_members to authenticated;

-- Users may only edit these columns. Ownership/status can never be changed from the browser.
grant update (full_name, phone, preferred_language) on public.profiles to authenticated;

grant update (
  name, phone, address, village_city, district, state, pincode, gstin,
  pesticide_license_no, fertilizer_license_no, seed_license_no, invoice_prefix
) on public.shops to authenticated;
