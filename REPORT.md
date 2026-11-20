# When five failed sign-ins deserve a closer look

**Musa Dadiri · SSH detection lab · November 20, 2026**

Five failed sign-ins can tell two very different stories. In my first Azure exercise, a student account failed five times and then signed in at 09:07. At first I read that as someone finally guessing the password. When I looked back at the records, all I really knew was that the sixth attempt succeeded. That gap shaped my project: I wanted an alert that could tell me which attempts to investigate, and a way to check exactly why it fired.

## Testing the idea in Azure

I started with two fictional accounts in Azure Log Analytics. Counting failures gave me a starting point, but it treated five mistakes spread across an hour the same as five in four minutes. I used KQL to count failures in a ten-minute window for each account. Moving the fifth failure from minute four to minute eleven made the match disappear.

Then I gave both accounts five quick failures. One scenario represented password guessing and the other represented a user entering an old password. The timing rule flagged both. Filtering for devices labeled “New” removed the old-password case. Changing the guessing case to “Usual” made that one disappear too. Device familiarity helped explain an alert, but treating it as a requirement could hide a case I wanted to investigate. I kept that experiment in the KQL portion of the project.

## Following an alert through Wazuh

I installed Wazuh on an Ubuntu VM in Azure Lab Services. Port 443 was already in use, so I put the dashboard on 8443. To create a repeatable test, I wrote SSH-style messages to the VM's journal with logger. I could choose the username, source address and sequence ahead of time, then compare the raw messages with the alerts that appeared.

I checked the built-in rules first. A single nonexistent username produced rule 5710. A burst of ten failures produced nine 5710 alerts and one level-10 alert under rule 5712. In the ordinary-mistake scenario, I saw a failed password (5760) followed by a successful authentication (5715). That gave me a reference for reading the custom results.

## The threshold test

My level-8 rule, 100105, looks for five matches of 5710 within 600 seconds for the same decoded source username. The source IP helped me find each test in the dashboard. The username was the field my rule used to group the failures.

For the six-event test, I numbered the source ports from 52001 through 52006. The dashboard returned six hits for 192.0.2.32. Four carried rule 5710 and two carried my rule. Opening the two custom alerts showed ports 52005 and 52006. Those numbers let me identify the exact inputs even though several dashboard timestamps looked alike.

For the control, I sent five failures from one fresh IP, alternating between two usernames so the counts ended at three and two. All five events appeared under rule 5710. Seeing every input mattered: an empty search for my custom rule alone would also have been possible if the messages had never arrived.

## The run I still can't explain

An earlier live run produced five ordinary alerts for one username. I checked the manager's start time, looked at the decoded usernames, and read the local alerts file. All five events were there, with distinct ports, and all five carried 5710. When I replayed the journal messages through wazuh-logtest, rule 100105 matched. I couldn't establish why the first live run differed. The later six-event run confirmed that the live rule could fire on the fifth input. It leaves the earlier discrepancy open.

## What I would investigate next

Given a similar alert, I'd first check the username, source and sequence, then look for a subsequent successful login and verify whether it belonged to a real account. This rule covers failures involving a nonexistent username. A guessed password against a valid account, or failures spread across many names, would need separate detection logic. The lab gave me evidence about those specific test cases. Estimating how often the rule raises false alarms in everyday use would take a larger set of labeled sign-ins.
