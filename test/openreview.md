---
layout: act
---

# Notes on using OpenReview (written after the event)

[OpenReview](https://openreview.net/) is a free conference management system. The default workflow is a little different from what is common in theoretical computer science venues, such as provided by easychair and hotcrp. But we found it was quite easy to adapt it. Moreover OpenReview provides free email support, and a [Python API](https://openreview-py.readthedocs.io/en/latest/) so that you can tailor things quite considerably and fairly easily. Here are some notes from the ACT 2023 PC chairs  in case they are useful for future events. 

We used OpenReview in single-blind mode (PC members do see author names). 

Do get in touch with Sam Staton if anything is unclear. 

## Main differences compared to familiar workflows

* OpenReview does automatic conflict-of-interest detection, which works well if it knows fully about the PC members and authors, but less well if either didn't properly complete their profiles, or where there is custom information (e.g. people about to move jobs), or where PC members are not on DBLP.  You can manually input conflicts-of-interest via the Python API, see below. 

* OpenReview doesn't by default allow PC members to invite non-PC subreviewers to join the discussion. The PC chairs have to add each subreviewer manually, if they want to have access to the OpenReview discussion (although there is probably a Python API route too). Alternatively of course the PC members can upload a review on behalf of the subreviewer, and communicate the discussion separately (e.g. by email) as needed.

* Many venues allow PC members to view discussion on papers that they were not assigned, provided there is not a pending review. In OpenReview this is not the default. The easiest options are (a) allow PC members to view all discussions (including conflict-of-interest) or (b) none. This was fixable by creating viewer groups for each paper, which included those PC members who are not conflicted with that paper. OpenReview support kindly created these groups for us. It is presumably also possible to do this via the Python API. This should be done once most of the reviews are in, as the discussion phase is entered (it might not be possible to automatically start this). 

## Record of Python experience

Our experience was with OpenReview v1. A couple of points to note.

* OpenReview does have a [dev server](https://dev.openreview.net/), which you can use for trying out configurations and seeing what happens when you run processes. It took a little while to set up an account on this, and you'll need at least two accounts for testing permissions. If you are using non-standard OpenReview workflows or experiment with the python API then I would recommend to do this earlier on in the process and not right before you are about to do something urgent and important. 

* Initially I was worried that I might accidentally permanently delete or overwrite papers or reviews. But actually I needn't have worried, it seems that most things in the OpenReview API are reversible. For example, edges are never deleted, you just assign them a "deletion date". 

### Connecting: 

```py 
import openreview
client = openreview.Client(baseurl='https://api.openreview.net', username='redacted', password='redacted')
```

### Download all paper pdfs:

```py
for note in notes:
    if(note.content.get("pdf")):
        f = client.get_attachment(note.id,'pdf')
        with open(f'./act2023papers/paper{note.number}.pdf','wb') as op: 
            op.write(f)
```

### Extract a tab-separated list of papers and bids:

```py
print(client.get_edges_count(invitation='AppliedCategoryTheory.org/ACT/2023/Conference/Reviewers/-/Bid'))
papers = openreview.tools.iterget_notes(
    client, 
	invitation='AppliedCategoryTheory.org/ACT/2023/Conference/-/Submission',
    )
for p in papers:
    bids = openreview.tools.iterget_edges(
        client,
		invitation='AppliedCategoryTheory.org/ACT/2023/Conference/Reviewers/-/Bid',
        head=p.id
        )
    for b in bids:
        print(p.content['title'],"\t",b.head,"\t",b.tail,"\t",b.label)
```

### Extract a tab-separated list of papers and conflicts-of-interest

```py
papers = openreview.tools.iterget_notes(
    client,
    invitation='AppliedCategoryTheory.org/ACT/2023/Conference/-/Submission',
    )
for p in papers:
    conflicts = openreview.tools.iterget_edges(
        client,
        invitation='AppliedCategoryTheory.org/ACT/2023/Conference/Reviewers/-/Conflict',
        head=p.id
        )
    print(p.number,"\t",p.content['title'],"\t",p.content['authors'],"\t",[c.tail for c in conflicts if c.weight==-1 ])
```


### Delete a conflict of interest

```py
edges = client.get_edges(invitation = 'AppliedCategoryTheory.org/ACT/2023/Conference/Reviewers/-/Conflict',head='paperid',tail='~userid1')
for edge in edges:
    print(edge)
    edge.ddate = 1664467200000
    client.post_edge(edge)
```

### Add a conflict of interest

```py
# add conflict
client.post_edge(openreview.Edge(
   invitation = 'AppliedCategoryTheory.org/ACT/2023/Conference/Reviewers/-/Conflict',
   label = "Custom Conflict", 
   weight = -1, 
   head = "paperid", 
   tail = "~userid1",
   signatures = [
"AppliedCategoryTheory.org/ACT/2023/Conference"
    ],
   readers = [
    "AppliedCategoryTheory.org/ACT/2023/Conference",
      "~userid1"
   ],
   writers = [
    "AppliedCategoryTheory.org/ACT/2023/Conference"
]))
```

(We first extracted the list of known COIs, put it on a google sheet, asked PC members to annotate corrections on the google sheet, and then updated the system using the add/remove scripts above. We could have done this entirely automatically, but there were sufficiently few changes that we just added/removed them one by one using the above two scripts, and anyway it doesn't hurt to double-check COI changes.)


### Extract a tab-separated list of papers and authors who are on the PC

```py
papers = openreview.tools.iterget_notes(
    client,
    invitation='AppliedCategoryTheory.org/ACT/2023/Conference/-/Submission',
    )

for p in papers:
    conflicts = openreview.tools.iterget_edges(
        client,
        invitation='AppliedCategoryTheory.org/ACT/2023/Conference/Reviewers/-/Conflict',
        head=p.id
        )
    authorcois =list(set(p.content['authorids']) & set([c.tail for c in conflicts if c.weight==-1 ]))
    if authorcois != []:
        print(p.number,"\t",p.content['title'],"\t",authorcois)
```		
	
### Extract a list of the authors on the PC.

```py
papers = openreview.tools.iterget_notes(
    client,
    invitation='AppliedCategoryTheory.org/ACT/2023/Conference/-/Submission',
    )
authorsOnPc = []
for p in papers:
    conflicts = openreview.tools.iterget_edges(
        client,
        invitation='AppliedCategoryTheory.org/ACT/2023/Conference/Reviewers/-/Conflict',
        head=p.id
        )
    authorsOnPc += list(set(p.content['authorids']) & set([c.tail for c in conflicts if c.weight==-1 ]))

print(sort(set(authorsOnPc)))
```
