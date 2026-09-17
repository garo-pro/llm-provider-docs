# Assign a program to workspaces in Claude Console

Anthropic offers several verification programs, such as the Cyber Verification Program, or access to models that might not be generally available. In order to gain access to these programs, go to our **[Verification Portal](https://portal.anthropic.com/)** to see what programs are available to you, and apply.

Once you’ve applied and been approved for a program, Anthropic issues a “program” to your organization. In order for it to be used, you must assign it to a group of people within the organization. In the Claude Console, a program applies to workspaces, either automatically (for programs like the Cyber Verification Program) or by assignment.

This article covers how to enable programs for the Console.

## Before you start

- Your organization must already have a grant. Grants appear only after Anthropic issues one to your organization. To apply to a specific program, go to our **[Verification Portal](https://portal.anthropic.com/)** to see what programs are available.

- In the Console, you need to be an organization Admin. Other roles cannot view or manage grants.

## Give a Console workspace access

In the Console, programs are issued to your organization and apply to workspaces. Some programs, such as the Cyber Verification Program, apply automatically to every workspace that meets their requirements. Others need workspaces assigned. A program only applies to API traffic from workspaces that meet its requirements.

**Follow these steps:**

1. **[Sign in to the Console](https://platform.claude.com/)** as an organization Admin. Go to **[Organization settings > Programs](https://platform.claude.com/settings/organization/programs)**. The program card shows whether it applies automatically or needs workspaces assigned.

  ![](https://downloads.intercomcdn.com/i/o/lupk8zyo/2642744587/d4584e035604f3b7c08afa53a1c6/ee1183ff-e591-4484-a989-1f754245d39c?expires=1789625700&amp;signature=cee9f60cc1756ad1623d80ac3745423886b6e0059d47da4d9099bd849e9f7eee&amp;req=diYjFM56mYRXXvMW1HO4zT%2FymECBFgygktHcEoeKC4eVh5TtYU8K6qc5d2Pt%0AQxfq%0A)

2. Select the program to open its page. The **Workspaces** table shows each workspace's status. A workspace marked with an issue does not meet a requirement yet.

  ![](https://downloads.intercomcdn.com/i/o/lupk8zyo/2642745562/253e55a3292b35f728fb5dc89fb2/0878a8a9-dce5-4df2-9826-3796605b52a0?expires=1789625700&amp;signature=fc3763d092056c8c2acce31ed8fdeb1aec85edea67bfb83f30d530644d053ddc&amp;req=diYjFM56mIRZW%2FMW1HO4zc116gduQlW2MCr%2B42fbmkZMzLRJ1mKUuj4cz33X%0AnYsA%0A)

Hover over the issue to see which requirement is not met.

  ![](https://downloads.intercomcdn.com/i/o/lupk8zyo/2642746466/c49291119729e99f4dba8ec924e4/3f802c0e-7fbc-4e80-935a-05da58f65bde?expires=1789625700&amp;signature=aeed6bd573d62fa712deca01ebda7fa799c04b2ed64c6d5f8b4743ff3b0b362a&amp;req=diYjFM56m4VZX%2FMW1HO4zaveae9klHjHVPpeIJbmktRgaBi1I%2FMpoV%2F1N%2FXh%0A%2Fn%2Bh%0A)

3. To give a workspace access, make it meet the requirements. Open the workspace, select "Manage," then "Programs," and check the **Qualifications** panel.

  ![](https://downloads.intercomcdn.com/i/o/lupk8zyo/2642768117/1304e6b1350fc9bd88c4238a00e3/db606eb5-39d5-4309-a5a9-ee33847fc233?expires=1789625700&amp;signature=8c58803deb507d0041d3376031835a3fbfd5e131913d7918e4391d66bcb2441c&amp;req=diYjFM54lYBeXvMW1HO4zTU0lNKWKUNA9BWcjfiNKI11XSekHTPBBhp%2Fiq8B%0AptIJ%0A)

4. Fix the requirement. For the Cyber Verification Program, turn on data retention under Manage, then Privacy controls. Then select "Rerun."

  ![](https://downloads.intercomcdn.com/i/o/lupk8zyo/2642746995/87151a11687a9c631b7a9d681390/d40a6c12-283d-4b3b-b6d6-9f631a73e7c0?expires=1789625700&amp;signature=63bde9ea44b32669583c6ad0da33f6d47d444c5fab6a82346b6d456fafaa7af1&amp;req=diYjFM56m4hWXPMW1HO4zQfcHTKo7Xss9apHi%2BiM8oikuzbiPv2qv2ZviZEM%0AFfm7%0A)

5. The program shows **Active** for the workspace.

  ![](https://downloads.intercomcdn.com/i/o/lupk8zyo/2642747200/a18bdccde474c9f4eba371cf6050/b0e9d5e3-1e5f-4f27-b682-5684084f92e8?expires=1789625700&amp;signature=b7af0bbfcc2cb48b8a29dd2e05d4ca83150198a7cbe7c2ebab1511f0152aa674&amp;req=diYjFM56moNfWfMW1HO4zaUR8q9h9Ps%2FfTukdAE3MWssX7JMdlhozxE7gj0z%0Arcab%0A)

## Troubleshooting

- **The Grants page is missing.** Your organization does not have a grant yet, or you are not an organization Admin. Contact your Anthropic account team or your admin.

- **The workspace shows as inactive.** Open the workspace, select "Manage," then "Programs," and check the **Qualifications** panel for an unmet requirement. Fix each unmet requirement and try again.

- **The grant is over its seat limit.** Some programs have a seat cap. Assigned workspaces lose access until your organization is back under the limit. Reduce the number of members counted toward the grant, then check again.

- **You are trying to use the default Console workspace.** Some programs don't allow the program to be assigned to the default workspace. If the default workspace isn’t working, assign a different workspace or create a new one.