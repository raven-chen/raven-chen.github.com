---
layout: post
title: "Raise or render 404 ?"
description: "Raise RecordNotFound or render 404 explicitly"
category:
tags: [rails]
---


We met a problem about when to raise RecordNotFound and let Rails handle rest things and when to render 404 explicitly. My colleague said:

I think in the case where there is only one condition "find this thing", it's fine to do that. But if the authorisation depends on more, for example: "the API client needs scope X to see this", then having the two conditions close to each-other is clearer. So in this example:

<pre>
service = user.outside_services.find params[:id] # <= could also raise RecordNotFound, and result in 404, but not very visible from the code
if client_has_scope_for service
  render json: service
else
  head status: :unauthorized
end
</pre>

vs

<pre>
service = user.outside_services.find_by_id params[:id]
if !service
  head :not_found
elsif !client_has_scope_for service
  head status: :unauthorized
else
  render json: service
end
</pre>

My understanding is `If we have to use 'if' anyway, then use it for all conditions`. All logic is visible and easier to understand.
