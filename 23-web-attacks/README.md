# Web Attacks

> Status: 🟨 In progress
> Date started: 2026-10-26
> Date finished: 

## 1. Goal
Advanced web attacks (HTTP verb tampering, IDOR, XXE, and related logic flaws).

## Mass Assignment (reference)
Framework binds user-supplied request params straight onto a model/object without a whitelist, so an attacker adds extra params (e.g. `admin=true`, `confirmed=1`) to set fields they shouldn't control → **privilege escalation / approval bypass / unauthorized data change**.

**Exploit = parameter tampering.** Add unexpected fields to the request body:
```http
POST /register HTTP/1.1
Content-Type: application/x-www-form-urlencoded

username=newuser&password=pw123&confirmed=1     # sets an attribute the app meant to control
```
Rails example: `User.new(params[:user])` + sending `admin=true`.

**Fix = whitelist the fields:**
- Rails strong params: `params.require(:user).permit(:username, :email)`
- Laravel: `protected $fillable = ['username','email'];` (not `$guarded = []`)
- Express/Sequelize: build the object from named fields, not `req.body` wholesale

**Finding it:** look for register/profile/update endpoints, compare the object's full field list (from source or responses) against what the form sends, then try adding the privileged fields. Real cases: GitHub 2012, Shopify admin panel.

## My notes / lab output
Filled in as I run these on lab targets.

## 2. Setup
- Attack box: (Pwnbox / Kali VM)
- Target(s): `<TARGET_IP>` (placeholders only, no real lab IPs)

## 3. Commands I used
Each block says which machine it ran on.

```bash
# [ATTACK BOX]

```

```bash
# [TARGET]

```

## 4. Confirmed (what worked)
- 

## 5. Tried, didn't work (and why)
- 

## 6. Open questions / revisit
- 

## 7. Cheat sheet lines
Copied into `cheatsheets/`:
- 
