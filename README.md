# kdl-merge
Merge for KDL documents.

## What is this?
This is my attempt to create support for merging KDL documents.

KDL (a Cuddly Document Language): https://kdl.dev/

## Status
The project is in draft state.

The specification is being worked on, but no code has been written yet.

## The problem
Merging two KDL documents is not trivial:
- **Multiple nodes with the same name can exists in the same document.**
  Should nodes with same name in both documents co-exist? Or should the new node overwrite the old node?
- Is two nodes are merged, should args/props/child nodes be overwritten? Or should they be merged, and how should the merge work?
- Should merging two KDL documents be able to delete nodes?
  KDL supports empty nodes (has a name but no value). So having a node with value null in the new document may/may not indicate that the node should be deleted.
- And more...

So my solution is to let the user define the rules of the merge behavior explicitly, instead of implementing a default merge behavior.

## Current Draft
### CLI program
A CLI program with the following usage:
```
USAGE: kdl-merge [-r RULEFILE] [-o NEWFILE] ORIGFILE MERGEFILE
```
An original KDL document ORIGFILE is merged with another KDL document MERGEFILE using the rules in a third KDL document RULEFILE. The merge result is written to NEWFILE.
- IF RULEFILE is included, then the merge rule is expected to be found in that file in a node called `kdl-merge`.
- If RULEFILE is omitted, the merge rules is instead looked for in MERGEFILE in a slashdash:ed node called `/-kdl-merge`.
- If NEWFILE is omitted, the merge result is written to stdout.
- If any of ORIGFILE, MERGEFILE, or RULEFILE, is `-` it is read from stdin (only one `-` is allowed).

### Rules Definition
Here is an example RULEFILE.
- Contains an EDITION for specifying which edition of the kdl-merge spec the file adhedes to.
- Contains RULES for how a merge should behave.
- Contains optional ASSERTS for proving that the rules work as intended.

```kdl
kdl-merge {
    // EDITION
    // The "edition" of the kdl-merge spec.
    // I imagine future editions to be called: "alpha" "beta" "preview" "2025" "2025-successor-preview" "2026"
    // New versions of an existing edition are not allowed to have breaking changes.
    edition "draft"

    // RULES
    // Default rule. Create the node if no other rules apply:
    rule { node "create"; }
    // Default match rule. Replace the node if no other rules apply:
    rule { node "modify"; args "overwrite"; props "overwrite"; nodes "overwrite"; if_match "name"; }

    // Delete rules. How to specify if a node shall be deleted, or if any of args/props/child nodes shall be cleared:
    rule { node "delete";                if_match "name"; if_new_is_exactly         null;         }
    rule { node "modify"; args  "clear"; if_match "name"; if_new_args_is_exactly    null null;    }
    rule { node "modify"; props "clear"; if_match "name"; if_new_props_is_exactly   null=null;    }
    rule { node "modify"; nodes "clear"; if_match "name"; if_new_nodes_is_exactly { null null; }; }

    rule { node "modify"; args "subtract"; if_match "name"; if_new_args_begins_with null; if_new_args_is_not_exactly null null; }

    // Error rule. How to specify nodes that shall fail the whole merge operation if they exist in MERGEFILE:
    rule {
        node "error" msg="Props containing null=null cannot contain any other key values."
        if_match "name"; if_new_props_has null=null; if_new_props_is_not_exactly null=null
    } 

    // Merge rules. When node shall be kept and args/props should be merged separately:
    rule { node "modify"; props "merge" { del_values null; }; if_match "name";        if_new_args_is_exactly "merge"; if_both_has "props"; if_both_has_no "nodes"; if_old_has_no "args"; }
    rule { node "modify"; nodes "merge";                      if_match "name" "args";                                 if_both_has "nodes"; if_both_has_no "props";                       }

    // Misc rule. Match on something other than node name and overwrite name if a match is found:
    rule { node "modify"; name "overwrite"; args "overwrite"; props "overwrite"; if_path "binds"; if_match "nodes"; }

    // ASSERTS
    // Each assert node contains three node children: orig (ORIGFILE), merge (MERGEFILE), and new (NEWFILE).
    // Assertion fails if "orig" merged with "merge" does not match "new" (the order of the resulting nodes is ignored).
    assert "all rules work as they should" {
        orig {
            example        "node"  that="is"    { not_modified; }
            modified       "arg"   key="value"  { child_node; }
            deleted        "arg"   key="value"  { child_node; }

            args_cleared   "arg"   key="value"  { child_node; }
            props_cleared  "arg"   key="value"  { child_node; }
            nodes_cleared  "arg"   key="value"  { child_node; }
            all_cleared    "arg"   key="value"  { child_node; }
            
            args_subtract  "a" "b" "c" "d" "e"

            props_merge  key1=value1 key2=value2 key3=value3
            nodes_merge             { child_node; }
            nodes_merge  "subname"  { child_node; }

            binds {
                ctrl+n  { new_window; }
                super+t { run "xterm"; }
            }
        }
        merge {
            created   "arg"    key="value"     { child_node; }
            modified  "blarg"  krum="failure"  { different_child_node; }
            deleted    null

            args_cleared    null null
            props_cleared   null=null
            nodes_cleared { null null; }
            all_cleared     null null  null=null  { null null; }
            
            args_subtract  null "b" "c" "e"

            props_merge  "merge"    key1=null key3=another_value
            nodes_merge             { new_child_node; }
            nodes_merge  "subname"  { new_child_node; }
            
            binds {
                ctrl+shift+n { new_window; }
                ctrl+f       { toggle_fullscreen; }
            }
        }
        new {
            created   "arg"    key="value"     { child_node; }
            example   "node"   that="is"       { not_modified; }
            modified  "blarg"  krum="failure"  { different_child_node; }

            args_cleared           key="value"  { child_node; }
            props_cleared  "arg"                { child_node; }
            nodes_cleared  "arg"   key="value"
            all_cleared
            
            args_subtract  "a" "d"

            props_merge  key2=value2 key3=another_value
            nodes_merge             { child_node; new_child_node; }
            nodes_merge  "subname"  { child_node; new_child_node; }

            binds {
                ctrl+shift+n { new_window; }
                ctrl+f       { toggle_fullscreen; }
                super+t      { run "xterm"; }
            }
        }
    }

    // TODO:
    assert_error "invalid nodes in new document" {
        old {}
        new {}
        errors {}
    }
}
```
