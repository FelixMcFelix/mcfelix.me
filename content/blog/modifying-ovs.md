+++ 
title = "Custom OpenVSwitch Actions"
date = 2017-11-23T14:54:55Z
description = ""
image = ""
tags = ["C", "Open vSwitch", "SDN"]
categories = ["Development", "Tutorial", "Networks"]
draft = true
+++

GOAL: Add probabilistic drop functionality as an action to OpenVSwitch.
(inspired by [this mailing list entry](https://mail.openvswitch.org/pipermail/ovs-discuss/2015-May/037560.html))

notes:

in datapath/linux/compat/include/linux/openvswitch.h:
	at "enum ovs_action_attr{ ... }"
	add your new enum entry! Name it well!
```c
enum ovs_action_attr {
	/* ... */

	/* after ifndef _KERNEL. the equals is thus ABSOLUTELY NECESSARY */

	OVS_ACTION_ATTR_PROBDROP=22,     /* unit32_t, probability in [0,2^32 -1] */
	/* ... */
}
```
(also probably modify the comment above it?)

in lib/packets.h:
	prototype for...
```c
bool prob_drop(uint32_t prob);

#endif /* packets.h */

```

in lib/packets.c:
	something (???)
```c
/* Ask for a random number.
   "p" is the amount we should let through, here true means drop,
   false means let it pass on */
bool
prob_drop(uint32_t prob)
{
	unsigned int roll_i;
	random_bytes(&roll_i, sizeof(roll_i));
	return roll_i > prob;
}
```

in datapath/actions.c: (can't use rand() in the kernel).
```c
static bool prob_drop(uint32_t prob)
{
	return prandom_u32() > prob;
}

static int do_execute_actions(/* ... */)
{
	/* ... */
	switch (nla_type(a)) {
	/* ... */
	case OVS_ACTION_ATTR_PROBDROP:
		/* No need to free, taken care of for us
		   This function just reads the attribute to
		   know if we should drop. */
		if(prob_drop(nla_get_u32(a)))
		{
			while (rem) {
				a = nla_next(a, &rem);
			}
		}
		break;
	}
	/* ... */
}
```

((ATTEMPT 2))

to help:
openvswitch/ofp-parse.h
```c
// add to block of relateds
char *str_to_f(const char *str, float *valuep) OVS_WARN_UNUSED_RESULT;
```

openvswitch/ofp-parse.c
```c
/* Parses 'str' as a 32-bit float into '*valuep'.
 *
 * Returns NULL if successful, otherwise a malloc()'d string describing the
 * error.  The caller is responsible for freeing the returned string. */
char * OVS_WARN_UNUSED_RESULT
str_to_f(const char *str, float *valuep)
{
	char *tail;
	float value;

	if (!str[0]) {
		return xstrdup("missing required numeric argument");
	}

	errno = 0;
	value = strtof(str, &tail);
	if (errno == EINVAL || errno == ERANGE || *tail) {
		return xasprintf("invalid numeric format %s", str);
	}
	*valuep = value;
	return NULL;
}
```

(Ignore last two, keep for posterity atm)

in include/openvswitch/ofp-actions.h:
```c
#define OFPACTS
	/* ... */
	OFPACT(GOTO_TABLE,      ofpact_goto_table,  ofpact, "goto_table") \
	OFPACT(PROBDROP,        ofpact_probdrop,    ofpact, "probdrop")

/* ..., after "struct ofpact_decap { ... }" */

/* OFPACT_PROBDROP.
 *
 * Used for OFPAT_PROBDROP */
struct ofpact_probdrop {
	struct ofpact ofpact;
	uint32_t prob;           /* Uint probability, "covers" 0->1 range. */
};
```

in lib/ofp-actions.c:
```c
enum ofp_raw_action_type {
	/* ... */

	/* NX1.3+(47): struct nx_action_decap, ... */
	NXAST_RAW_DECAP,

	/* OF1.0+(29): uint32_t. */
	OFPAT_RAW_PROBDROP,

	/* ... */
}

/* ... */

struct ofpact *
ofpact_next_flattened(const struct ofpact *ofpact)
{
	switch (ofpact->type) {
		/* ... */
		case OFPACT_PROBDROP:
			return ofpact_next(ofpact);
	}
	/* ... */
}

/* ... */

static bool
ofpact_is_set_or_move_action(const struct ofpact *a)
{
	switch (a->type) {
	/* ... */
	case OFPACT_PROBDROP:
		return false;
	/* ... */
	}
}

/* ... */

static bool
ofpact_is_allowed_in_actions_set(const struct ofpact *a)
{
	switch (a->type) {
	/* ... */
	case OFPACT_PROBDROP:
		return true;
	/* ... */
	}
}

/* ... */

enum ovs_instruction_type
ovs_instruction_type_from_ofpact_type(enum ofpact_type type)
{
	switch (type) {
	/* ... */
	case OFPACT_PROBDROP:
	default:
		return OVSINST_OFPIT11_APPLY_ACTIONS;
	/* ... */
	}
}

/* ... */

static enum ofperr
ofpact_check__( /* ... */ )
{
	/* ... */
	switch (a->type) {
	/* ... */
	case OFPACT_PROBDROP:
		return 0;
		/* My method needs no checking. Probably. */
	}
}

/* ... */

static bool
ofpact_outputs_to_port(const struct ofpact *ofpact, ofp_port_t port)
{
	switch (ofpact->type) {
	/* ... */
	case OFPACT_PROBDROP:
	default:
		return false;
	}
}
```

Now, same file:
```c
/* ..., immediately after format_GOTO_TABLE */

/* Okay, the new stuff! */

/* Encoding the action packet to put on the wire. */
static void
encode_PROBDROP(const struct ofpact_probdrop *prob,
					enum ofp_version ofp_version OVS_UNUSED,
					struct ofpbuf *out)
{
	uint32_t p = prob->prob;

	put_OFPAT_PROBDROP(out, p);
}

/* Hmm. */
static enum ofperr
decode_OFPAT_RAW_PROBDROP(uint32_t prob,
							enum ofp_version ofp_version OVS_UNUSED,
							struct ofpbuf *out)
{
	struct ofpact_probdrop *op;
	op = ofpact_put_PROBDROP(out);
	op->prob = prob;

	return 0;
}

/* Helper for below. */
static char * OVS_WARN_UNUSED_RESULT
parse_prob(char *arg, struct ofpbuf *ofpacts)
{
	struct ofpact_probdrop *probdrop;
	uint32_t prob;
	char *error;

	error = str_to_u32(arg, &prob);
	if (error) return error;

	probdrop = ofpact_put_PROBDROP(ofpacts);
	probdrop->prob = prob;
	return NULL;
}

/* Go from string-formatted args into an action struct. */
static char * OVS_WARN_UNUSED_RESULT
parse_PROBDROP(char *arg,
					const struct ofputil_port_map *port_map OVS_UNUSED,
					struct ofpbuf *ofpacts,
					enum ofputil_protocol *usable_protocols OVS_UNUSED)
{
	return parse_prob(arg, ofpacts);
}

/* Used when printing info to console. */
static void
format_PROBDROP(const struct ofpact_probdrop *a,
					const struct ofputil_port_map *port_map OVS_UNUSED,
					struct ds *s)
{
	/* Feel free to use e.g. colors.param, colors.end around parameter names */
	ds_put_format(s, "probdrop:%"PRIu32, a->prob);
}
```

If you want more complex handling:
```c
/* Simple key-value walker. Useful for complex actions. */
char *key;
char *value;
while (ofputil_parse_key_value(&arg, &key, &value)) {
	if (!strcmp(key, "P_send")) {
		error = str_to_u32(value, &prob);
	} else {
		return xasprintf("invalid key '%s' in probdrop argument",
							key);
	}

	if (error) return error;
}
```

in build-aux/extract-ofp-actions:
```py
types['float'] =    {"size": 4, "align": 4, "ntoh": None,     "hton": None}
types['double'] =   {"size": 8, "align": 8, "ntoh": None,     "hton": None}
```

(ignore that ^^)

in ofproto/ofproto-dpif-xlate.c (This seems to prep and parse things on behalf of the very first functions we wrote.)
```c
/* Put this with the other "compose" functions. */
static void
compose_probdrop_action(struct xlate_ctx *ctx, struct ofpact_probdrop *op)
{
	uint32_t prob = op->prob;

	nl_msg_put_u32(ctx->odp_actions, OVS_ACTION_ATTR_PROBDROP, prob);
}

/* ... */

static void
do_xlate_actions( /* ... */ )
{
	switch (a->type) {
	/* ... */

	case OFPACT_PROBDROP:
		compose_probdrop_action(ctx, ofpact_get_PROBDROP(a));
		break;
	}
}

/* ... */

static bool
xlate_fixup_actions()
{
	/* ... */
	switch ((enum ovs_action_attr) type) {
	/* ... */
	case OVS_ACTION_ATTR_PROBDROP:
			ofpbuf_put(b, a, nl_attr_len_pad(a, left));
	/* ... */
	}
	/* ... */
}

/* ... */

/* No action can undo the packet drop: reflect this. */
static bool
reversible_actions(const struct ofpact *ofpacts, size_t ofpacts_len)
{
	const struct ofpact *a;

	OFPACT_FOR_EACH (a, ofpacts, ofpacts_len) {
		switch (a->type) {
		/*... */
		case OFPACT_PROBDROP:
			return false;
		}
	}
	return true;
}

/* ... */

/* PROBDROP likely doesn't require explicit thawing. */
static void
freeze_unroll_actions( /* ... */ )
{
	/* ... */
	switch (a->type) {
		case OFPACT_PROBDROP:
			/* These may not generate PACKET INs. */
			break;
	}
}

/* ... */

/* Naturally, don't need to recirculate since we don't change packets. */
static void
recirc_for_mpls(const struct ofpact *a, struct xlate_ctx *ctx)
{
	/* ... */

	switch (a->type) {
	case OFPACT_PROBDROP:
	default:
		break;
	}
}
```

in lib/odp-util.c: (for pretty-printing etc)
```c
static void
format_odp_action( /* ... */ )
{
	/* ... */
	switch (type) {
	/* ... */

	case OVS_ACTION_ATTR_PROBDROP: 
		ds_put_format(ds, "pdrop(%"PRIu32")", nl_attr_get_u32(a));
		break;

	/* ... */
	}
}

static int
odp_action_len(uint16_t type)
{
	/* ... */

	switch ((enum ovs_action_attr) type) {
	/* ... */
	case OVS_ACTION_ATTR_PROBDROP: return sizeof(uint32_t);
	}
}
```

in datapath/flow_netlink.c:
```c
static const u32 action_lens[OVS_ACTION_ATTR_MAX + 1] = {
	/* ... */
	[OVS_ACTION_ATTR_PROBDROP] = sizeof(u32),
};
```

in lib/dpif-netdev.c:
```c
static void
dp_execute_cb( /* ... */ )
{
	/* ... */
	switch ((enum ovs_action_attr)type) {
	/* ... */
	case OVS_ACTION_ATTR_PROBDROP:
		OVS_NOT_REACHED();
	}
}
```

in lib/dpif.c:
```c
static void
dpif_execute_helper_cb( /* ... */ )
{
	/* ... */
	switch ((enum ovs_action_attr)type) {
	/* ... */
	case OVS_ACTION_ATTR_PROBDROP:
		OVS_NOT_REACHED();
	}
}
```

in lib/odp-execute.c: (for dpdk?)
```c
static bool
requires_datapath_assistance(const struct nlattr *a)
{
	enum ovs_action_attr type = nl_attr_type(a);

	switch (type) {
	/* ... */
	case OVS_ACTION_ATTR_PROBDROP:
		return false;
	/* ... */
	}
}

/* ... */

void
odp_execute_actions( /* ... */ )
{
	/* ... */
	switch ((enum ovs_action_attr)type) {
	/* ... */
	case OVS_ACTION_ATTR_PROBDROP: {
		size_t i;
		const size_t num = dp_packet_batch_size(batch);

		DP_PACKET_BATCH_REFILL_FOR_EACH (i, num, packet, batch) {
			if (!prob_drop(nl_attr_get_u32(a))) {
				dp_packet_batch_refill(batch, packet, i);
			} else {
				dp_packet_delete(packet);
			}
		}
		break;
	}

	}
}
```

Okay: now we're suffering compiler errors.

ofproto/ofproto-dpif-ipfix.c:
```c
void
dpif_ipfix_read_actions( /* ... */ )
{
	/* ... */
	switch (type) {
	/* ... */
	case OVS_ACTION_ATTR_PROBDROP:

		break;
	}
}
```

ofproto/ofproto-dpif-ipfix.c:
```c
void
dpif_sflow_read_actions( /* ... */ )
{
	switch (type) {
	/* ... */
	case OVS_ACTION_ATTR_PROBDROP:
		/* Ignore sFlow for now, unless needed. */
		break;
	}
}
```

Now: go through all [this crap](http://docs.openvswitch.org/en/latest/intro/install/general/) and [even this](https://github.com/mininet/mininet/wiki/Installing-new-version-of-Open-vSwitch) to install. Then how do we test? Mininet!

For the love of god, debug in gdb. ARGH.

If shit starts breaking for no reason, check `dmesg`. Huh, "netlink: Flow actions may not be safe on all matching packets." Whoops, guess the fucking compiler never told you about this one case switch HUH BUDDY.

In datapath/flow_netlink.c
```c
static int __ovs_nla_copy_actions( /*...*/ )
{
	/* ... */
	switch (type) {
	/* ... */
	case OVS_ACTION_ATTR_PROBDROP:
		/* Finalest sanity checks in the kernel. */
		break;
	/* ... */
	}
	/* ... */
}
```