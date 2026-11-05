---
slug: quiz-inheritance
id: vix6pwjfy0cz
type: quiz
title: 'Quiz: IAM inheritance'
teaser: Check your understanding of the hierarchy.
answers:
- Nothing, until the role is also granted on each project
- Only objects in buckets created after the grant
- Objects in every bucket in both projects (and any future project in that folder)
- Only the folder's metadata
solution:
- 2
difficulty: intermediate
timelimit: 300
lab_config:
  default_layout_sidebar_size: 0
enhanced_loading: null
---

Alice is granted roles/storage.objectViewer on the folder that contains the dev and staging projects. What can she read?
